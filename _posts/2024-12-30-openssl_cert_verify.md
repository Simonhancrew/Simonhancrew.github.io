---
title: openssl证书验证
date: 2024-12-30 17:30:00 +0800
categories: [Blogging, crypto, cert]
tags: [writing]
---

boringssl的ut写了一个非常good的例子

```cpp
TEST(X509Test, TestVerify) {
  //  cross_signing_root
  //         |
  //   root_cross_signed    root
  //              \         /
  //             intermediate
  //                |     |
  //              leaf  leaf_no_key_usage
  //                      |
  //                    forgery
  bssl::UniquePtr<X509> cross_signing_root(CertFromPEM(kCrossSigningRootPEM));
  bssl::UniquePtr<X509> root(CertFromPEM(kRootCAPEM));
  bssl::UniquePtr<X509> root_cross_signed(CertFromPEM(kRootCrossSignedPEM));
  bssl::UniquePtr<X509> intermediate(CertFromPEM(kIntermediatePEM));
  bssl::UniquePtr<X509> intermediate_self_signed(
      CertFromPEM(kIntermediateSelfSignedPEM));
  bssl::UniquePtr<X509> leaf(CertFromPEM(kLeafPEM));
  bssl::UniquePtr<X509> leaf_no_key_usage(CertFromPEM(kLeafNoKeyUsagePEM));
  bssl::UniquePtr<X509> forgery(CertFromPEM(kForgeryPEM));

  ASSERT_TRUE(cross_signing_root);
  ASSERT_TRUE(root);
  ASSERT_TRUE(root_cross_signed);
  ASSERT_TRUE(intermediate);
  ASSERT_TRUE(intermediate_self_signed);
  ASSERT_TRUE(leaf);
  ASSERT_TRUE(forgery);
  ASSERT_TRUE(leaf_no_key_usage);

  // Most of these tests work with or without |X509_V_FLAG_TRUSTED_FIRST|,
  // though in different ways.
  for (bool trusted_first : {true, false}) {
    SCOPED_TRACE(trusted_first);
    std::function<void(X509_VERIFY_PARAM *)> configure_callback;
    if (!trusted_first) {
      // Note we need the callback to clear the flag. Setting |flags| to zero
      // only skips setting new flags.
      configure_callback = [&](X509_VERIFY_PARAM *param) {
        X509_VERIFY_PARAM_clear_flags(param, X509_V_FLAG_TRUSTED_FIRST);
      };
    }

    // No trust anchors configured.
    ASSERT_EQ(X509_V_ERR_UNABLE_TO_GET_ISSUER_CERT_LOCALLY,
              Verify(leaf.get(), /*roots=*/{}, /*intermediates=*/{},
                     /*crls=*/{}, /*flags=*/0, configure_callback));
    ASSERT_EQ(
        X509_V_ERR_UNABLE_TO_GET_ISSUER_CERT_LOCALLY,
        Verify(leaf.get(), /*roots=*/{}, {intermediate.get()}, /*crls=*/{},
               /*flags=*/0, configure_callback));

    // Each chain works individually.
    ASSERT_EQ(X509_V_OK, Verify(leaf.get(), {root.get()}, {intermediate.get()},
                                /*crls=*/{}, /*flags=*/0, configure_callback));
    ASSERT_EQ(X509_V_OK, Verify(leaf.get(), {cross_signing_root.get()},
                                {intermediate.get(), root_cross_signed.get()},
                                /*crls=*/{}, /*flags=*/0, configure_callback));

    // When both roots are available, we pick one or the other.
    ASSERT_EQ(X509_V_OK,
              Verify(leaf.get(), {cross_signing_root.get(), root.get()},
                     {intermediate.get(), root_cross_signed.get()}, /*crls=*/{},
                     /*flags=*/0, configure_callback));

    // This is the “altchains” test – we remove the cross-signing CA but include
    // the cross-sign in the intermediates. With |trusted_first|, we
    // preferentially stop path-building at |intermediate|. Without
    // |trusted_first|, the "altchains" logic repairs it.
    ASSERT_EQ(X509_V_OK, Verify(leaf.get(), {root.get()},
                                {intermediate.get(), root_cross_signed.get()},
                                /*crls=*/{}, /*flags=*/0, configure_callback));

    // If |X509_V_FLAG_NO_ALT_CHAINS| is set and |trusted_first| is disabled, we
    // get stuck on |root_cross_signed|. If either feature is enabled, we can
    // build the path.
    //
    // This test exists to confirm our current behavior, but these modes are
    // just workarounds for not having an actual path-building verifier. If we
    // fix it, this test can be removed.
    ASSERT_EQ(trusted_first ? X509_V_OK
                            : X509_V_ERR_UNABLE_TO_GET_ISSUER_CERT_LOCALLY,
              Verify(leaf.get(), {root.get()},
                     {intermediate.get(), root_cross_signed.get()}, /*crls=*/{},
                     /*flags=*/X509_V_FLAG_NO_ALT_CHAINS, configure_callback));

    // |forgery| is signed by |leaf_no_key_usage|, but is rejected because the
    // leaf is not a CA.
    ASSERT_EQ(X509_V_ERR_INVALID_CA,
              Verify(forgery.get(), {intermediate_self_signed.get()},
                     {leaf_no_key_usage.get()}, /*crls=*/{}, /*flags=*/0,
                     configure_callback));

    // Test that one cannot skip Basic Constraints checking with a contorted set
    // of roots and intermediates. This is a regression test for CVE-2015-1793.
    ASSERT_EQ(X509_V_ERR_INVALID_CA,
              Verify(forgery.get(),
                     {intermediate_self_signed.get(), root_cross_signed.get()},
                     {leaf_no_key_usage.get(), intermediate.get()}, /*crls=*/{},
                     /*flags=*/0, configure_callback));
  }
}
```

verify的代码在下面

```cpp
static int Verify(
    X509 *leaf, const std::vector<X509 *> &roots,
    const std::vector<X509 *> &intermediates,
    const std::vector<X509_CRL *> &crls, unsigned long flags = 0,
    std::function<void(X509_VERIFY_PARAM *)> configure_callback = nullptr,
    int (*verify_callback)(int, X509_STORE_CTX *) = nullptr) {
  bssl::UniquePtr<STACK_OF(X509)> roots_stack(CertsToStack(roots));
  bssl::UniquePtr<STACK_OF(X509)> intermediates_stack(
      CertsToStack(intermediates));
  bssl::UniquePtr<STACK_OF(X509_CRL)> crls_stack(CRLsToStack(crls));

  if (!roots_stack ||
      !intermediates_stack ||
      !crls_stack) {
    return X509_V_ERR_UNSPECIFIED;
  }

  bssl::UniquePtr<X509_STORE_CTX> ctx(X509_STORE_CTX_new());
  bssl::UniquePtr<X509_STORE> store(X509_STORE_new());
  if (!ctx ||
      !store) {
    return X509_V_ERR_UNSPECIFIED;
  }

  if (!X509_STORE_CTX_init(ctx.get(), store.get(), leaf,
                           intermediates_stack.get())) {
    return X509_V_ERR_UNSPECIFIED;
  }

  X509_STORE_CTX_trusted_stack(ctx.get(), roots_stack.get());
  X509_STORE_CTX_set0_crls(ctx.get(), crls_stack.get());

  X509_VERIFY_PARAM *param = X509_STORE_CTX_get0_param(ctx.get());
  X509_VERIFY_PARAM_set_time(param, kReferenceTime);
  if (configure_callback) {
    configure_callback(param);
  }
  if (flags) {
    X509_VERIFY_PARAM_set_flags(param, flags);
  }

  ERR_clear_error();
  if (X509_verify_cert(ctx.get()) != 1) {
    return X509_STORE_CTX_get_error(ctx.get());
  }

  return X509_V_OK;
}
```

其中受信证书有两个办法

第一个是参照tls的

```cpp
bssl::UniquePtr<X509_STORE> store(X509_STORE_new());
// add trusted cert to store
X509_STORE_CTX_init(ctx.get(), store.get(), leaf,
                           intermediates_stack.get());
```

另外一种办法就是针对这个ctx设置受信任的证书

```cpp
X509_STORE_CTX_trusted_stack(ctx.get(), roots_stack.get());
```

在随后的`X509_verify_cert`中，这个代码上的逻辑是不一样的, 如果设置了trusted stack，会走下面的get_issuer.


```cpp
void X509_STORE_CTX_trusted_stack(X509_STORE_CTX *ctx, STACK_OF(X509) *sk) {
  ctx->other_ctx = sk;
  ctx->get_issuer = get_issuer_sk;
}

static int get_issuer_sk(X509 **issuer, X509_STORE_CTX *ctx, X509 *x) {
  *issuer = find_issuer(ctx, ctx->other_ctx, x);
  if (*issuer) {
    X509_up_ref(*issuer);
    return 1;
  } else {
    return 0;
  }
}

static X509 *find_issuer(X509_STORE_CTX *ctx, STACK_OF(X509) *sk, X509 *x) {
  size_t i;
  X509 *issuer;
  for (i = 0; i < sk_X509_num(sk); i++) {
    issuer = sk_X509_value(sk, i);
    if (ctx->check_issued(ctx, x, issuer)) {
      return issuer;
    }
  }
  return NULL;
}
```

另外如果往store里+trust证书的代码，会走下面的

```cpp
// Try to get issuer certificate from store. Due to limitations of the API
// this can only retrieve a single certificate matching a given subject name.
// However it will fill the cache with all matching certificates, so we can
// examine the cache for all matches. Return values are: 1 lookup
// successful.  0 certificate not found. -1 some other error.
int X509_STORE_CTX_get1_issuer(X509 **issuer, X509_STORE_CTX *ctx, X509 *x) {
  X509_NAME *xn;
  X509_OBJECT obj, *pobj;
  int idx, ret;
  size_t i;
  xn = X509_get_issuer_name(x);
  if (!X509_STORE_get_by_subject(ctx, X509_LU_X509, xn, &obj)) {
    return 0;
  }
  // If certificate matches all OK
  if (ctx->check_issued(ctx, x, obj.data.x509)) {
    *issuer = obj.data.x509;
    return 1;
  }
  X509_OBJECT_free_contents(&obj);

  // Else find index of first cert accepted by 'check_issued'
  ret = 0;
  CRYPTO_MUTEX_lock_write(&ctx->ctx->objs_lock);
  idx = X509_OBJECT_idx_by_subject(ctx->ctx->objs, X509_LU_X509, xn);
  if (idx != -1) {  // should be true as we've had at least one
                    // match
    // Look through all matching certs for suitable issuer
    for (i = idx; i < sk_X509_OBJECT_num(ctx->ctx->objs); i++) {
      pobj = sk_X509_OBJECT_value(ctx->ctx->objs, i);
      // See if we've run past the matches
      if (pobj->type != X509_LU_X509) {
        break;
      }
      if (X509_NAME_cmp(xn, X509_get_subject_name(pobj->data.x509))) {
        break;
      }
      if (ctx->check_issued(ctx, x, pobj->data.x509)) {
        *issuer = pobj->data.x509;
        X509_OBJECT_up_ref_count(pobj);
        ret = 1;
        break;
      }
    }
  }
  CRYPTO_MUTEX_unlock_write(&ctx->ctx->objs_lock);
  return ret;
}
```
