---
title: 模板方法"
date: 2022-10-12 14:10:00 +0800
categories: [Blogging]
tags: [writing]
---

## Template Method

在软件构建的过程中，整体的步骤流程是确定的A->B->C,假设其中A和B的流程完全明确，但是B的整体流程是需要自己实现的。


定义一个操作中的算法的骨架 (稳定)，而将一些步骤延迟(变化)到子类中。 Template Method使得子类可以不改变(复用)一个算法的结构即可重定义(overrided写)该算法的某些特定步骤。

一个简单的例子就是：

针对库的开发, 步骤1，3，5固定，步骤2，4是变化的

```
//程序库开发人员
class Library{
public:
	//稳定 template method
    void Run(){
        
        Step1();

        if (Step2()) { //支持变化 ==> 虚函数的多态调用
            Step3(); 
        }

        for (int i = 0; i < 4; i++){
            Step4(); //支持变化 ==> 虚函数的多态调用
        }

        Step5();

    }
	virtual ~Library(){ }

protected:
	
	void Step1() { //稳定
        //.....
    }
	void Step3() {//稳定
        //.....
    }
	void Step5() { //稳定
		//.....
	}

	virtual bool Step2() = 0;//变化
    virtual void Step4() =0; //变化
};
```
因此，可以在实现上继承这个lib
```
//应用程序开发人员
class Application : public Library {
protected:
	virtual bool Step2(){
		//... 子类重写实现
    }

    virtual void Step4() {
		//... 子类重写实现
    }
};

int main()
	{
	    Library* pLib=new Application();
	    lib->Run();

		delete pLib;
	}
}
```

### c++继承体系中的public, private, protected

| 继承方式 | 父类的public成员 | 父类的protected成员 | 父类的private成员 | 关系描述 | 
| ----     | ----             | ----                |   ---            | -------- | 
| public | 依然为public | 依然为protected | 不可见 |   |
| protected | 变为protected成员 | 变为protected成员 | 不可见 |  |  
| private | 变为private成员 | 变成private成员 | 不可见|  | 

protected存在的理由就是继承类中访问基类的protected成员。

引入保护成员的理由是：基类的成员本来就是派生类的成员，因此对于那些出于隐藏的目的不宜设为公有，但又确实需要在派生类的成员函数中经常访问的基类成员，将它们设置为保护成员，既能起到隐藏的目的，又避免了派生类成员函数要访问它们时只能间接访问所带来的麻烦。
