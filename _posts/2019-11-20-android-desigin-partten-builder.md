---
layout: post
title:  "设计模式之Builder模式"
date:   2019-11-20 00:00:00
catalog:  true
tags:
    - java
    - 设计模式
    - 建造者模式
---



## 1 概述

Builder模式也成为建造者模式，它允许用户在不知道内部构建细节的情况下，可以更精细地控制对象的构造流程。该模式是为了将构建复杂对象的过程和它的部件解耦，使得构建过程和部件的表示隔离。

很多开源库创建一个对象时，都是利用Builder模式来构建的，比较 okHttp开源库，如 构造Request对象 如下：

```java
Request request = new Request.Builder()
                .get()
                .url("https:www.baidu.com")
                .build();
```

## 2 使用场景

- 相同的方法，不同的执行顺序，产生不同的事件结果时
- 多个部件或零件，都可以装配到一个对象中，但是产生的运行结果又不相同时
- 产品类非常复杂，或者产品类中的调用顺序不同产生了不同的作用
- 当初始化一个对象特别复杂，如参数多，切很多参数都具有默认值时

上面创建Request实例就是最后一个使用场景，该使用场景也是应用Builder模式最多的。

## 3 UML类图

![uml-builder](/images/design-partten/uml-builder.PNG)

角色介绍：

- Product：产品的抽象类
- Builder：抽象Builder类，规范产品的组建，一般是由子类实现具体的组建过程
- ConcreteBuilder：具体的Builder类
- Director：监工类，负责统一组装

## 4 完整的Builder模式实现

```java
package com.vane.pattern;
//房子抽象类，即Product角色
public abstract class House {
	
	protected int mLength;
	protected int mWidth;
	protected int mHeight;
	protected String mStype;
	
	protected House() {}

	public void setLength(int mLength) {
		this.mLength = mLength;
	}

	public void setWith(int mWidth) {
		this.mWidth = mWidth;
	}

	public void setHeight(int mHeight) {
		this.mHeight = mHeight;
	}
	
	public abstract void setStype();
    
    @Override
	public String toString() {
		return "House [mLength=" + mLength + ", mWidth=" + mWidth + ", mHeight=" + mHeight + ", mStype=" + mStype + "]";
	}

}
```

```java
package com.vane.pattern;
//House的实现类，普通平房
public class OrdinaryHouse extends House {

	public OrdinaryHouse() {
	}

	@Override
	public void setStype() {
        mStype = "without garden";
	}
	
}
```

```java
package com.vane.pattern;
//House的实现类，别墅
public class Villa extends House {

	public Villa() {
	}

	@Override
	public void setStype() {
        mStype = "with a garden";
	}
	
}
```

```java
package com.vane.pattern;
//抽象Builder类，用来构建房子
public abstract class Builder {

	public abstract void buildLenght(int length);
	
	public abstract void buildWidth(int width);
	
	public abstract void buildHeight(int height);
	
	public abstract void buildStype();
	
	public abstract House create();

}
```

```java
package com.vane.pattern;
//Builder实现类，用来创建平房
public class OrdinaryHouseBuilder extends Builder {

	private House mHouse = new OrdinaryHouse();
	@Override
	public void buildLenght(int length) {
		mHouse.setLength(length); 
	}

	@Override
	public void buildWidth(int width) {
		mHouse.setWith(width);
	}

	@Override
	public void buildHeight(int height) {
        mHouse.setHeight(height);
	}

	@Override
	public void buildStype() {
        mHouse.setStype();
	}

	@Override
	public House create() {
		return mHouse;
	}

}
```

```java
package com.vane.pattern;
//Builder实现类，用来创建别墅
public class VillaBuilder extends Builder {

	private House mHouse = new Villa();
	@Override
	public void buildLenght(int length) {
		mHouse.setLength(length); 
	}

	@Override
	public void buildWidth(int width) {
		mHouse.setWith(width);
	}

	@Override
	public void buildHeight(int height) {
        mHouse.setHeight(height);
	}

	@Override
	public void buildStype() {
        mHouse.setStype();
	}

	@Override
	public House create() {
		return mHouse;
	}

}
```

```java
package com.vane.pattern;
//Director，负责构造House
public class Director {

	Builder mBuilder = null;

	public Director(Builder mBuilder) {
		this.mBuilder = mBuilder;
	}
	
	public void construct(int length, int width, int height) {
		mBuilder.buildLenght(length);
		mBuilder.buildWidth(width);
		mBuilder.buildHeight(height);
		mBuilder.buildStype();
	}
	
}
```

```java
package com.vane.pattern;
//测试类
public class BuilderClientTest {

	public static void main(String[] args) {
		Builder mBuilder = new OrdinaryHouseBuilder();
		Director director = new Director(mBuilder);
		director.construct(10, 5, 10);
		
		Builder mBuilder2 = new VillaBuilder();
		Director director2 = new Director(mBuilder2);
		director2.construct(100, 50, 20);
		System.out.println(mBuilder.create().toString());
		System.out.println(mBuilder2.create().toString());
	}

}
```

输出结果：

```logcat
House [mLength=10, mWidth=5, mHeight=10, mStype=without garden]
House [mLength=100, mWidth=50, mHeight=20, mStype=with a garden]
```

上述的例子中，通过具体的OrdinaryHouseBuilder和VillaBuilder来分别构建OrdinaryHouse和Villa对象，二Director封装了构建复杂产品对象的过程，对外隐藏构建细节。

## 5 简易版的Builder模式实现

上一小节的例子是完整的Builder模式，但通常在现实开发当中，通常会将Director角色省略，如开篇的Request构建就是这样：直接使用Request的静态内部类Builder来链式创建Request对象。接下来对于该种用法举个例子。

```java
public class Person {
	
	private String name = "Jack";
	private String sex = "Men";
	private int height = 170;
	
	private  Person(Builder builder) {
		this.name = builder.name;
        this.sex = builder.sex;
        this.height = builder.height;
	}
	
	public static class Builder {
		
		private  String name;
		private  String sex;
		private  int height;
		
		public Builder setName(String name) {
			this.name = name;
			return this;
		}

		public Builder setSex(String sex) {
			this.sex = sex;
			return this;
		}

		public Builder setHeight(int height) {
			this.height = height;
			return this;
		}
		
		public Person build() {
            return new Person(this);
        }
	}
	
	@Override
	public String toString() {
		return "Person[name=" + name + ", sex=" + sex + ", height=" + height + "]";
	}
}
```

上面的代码是利用Person类中的静态内部类Builder来创建一个Person实例，其他模块想创建一个Person很简单，如下：

```java
Person person = new Person.Builder()
				.setName("LiLy")
				.setSex("Women")
				.setHeight(165)
				.build();
		System.out.println(person.toString());
```

log打印如下：

```logcat
Person[name=LiLy, sex=Women, height=165]
```

其实传参数给Person的构造函数也能实现，上面的Person的构造函数私有化是为了不让通过new的方式来创建Person实例。通过构造函数的方式的缺点是一旦属性非常多，需要重载n多个构造器，而且各种构造器的组成都是在特定需求的情况下制定的，代码量多了不说，灵活性大大下降。客户端调用构造器的时候，需要传的属性非常多，可能导致调用困难，我们需要去熟悉每个特定构造器所提供的属性是什么样的，而参数属性多的情况下，我们可能因为疏忽而传错顺序。

## 6 Android源码中的Builder模式

在Android源码中，AlertDialog的创建就是利用Builder模式创建的，代码如下：

frameworks/base/core/java/android/app/AlertDialog.java

```java
public class AlertDialog extends Dialog implements DialogInterface {
    ...
    public static class Builder {
        private final AlertController.AlertParams P;
        public Builder(Context context, int themeResId) {
            P = new AlertController.AlertParams(new ContextThemeWrapper(
                    context, resolveDialogTheme(context, themeResId)));
        }
        ...
        //设置标题
        public Builder setTitle(@StringRes int titleId) {
            P.mTitle = P.mContext.getText(titleId);
            return this;
        }
        //设置内容信息
        public Builder setMessage(@StringRes int messageId) {
            P.mMessage = P.mContext.getText(messageId);
            return this;
        }
        //设置图标
        public Builder setIcon(@DrawableRes int iconId) {
            P.mIconId = iconId;
            return this;
        }
        ...
        //创建AlertDialog实例
        public AlertDialog create() {
            // Context has already been wrapped with the appropriate theme.
            final AlertDialog dialog = new AlertDialog(P.mContext, 0, false);
            P.apply(dialog.mAlert);
            dialog.setCancelable(P.mCancelable);
            if (P.mCancelable) {
                dialog.setCanceledOnTouchOutside(true);
            }
            dialog.setOnCancelListener(P.mOnCancelListener);
            dialog.setOnDismissListener(P.mOnDismissListener);
            if (P.mOnKeyListener != null) {
                dialog.setOnKeyListener(P.mOnKeyListener);
            }
            return dialog;
        }
        //显示创建的AlertDialog实例
        public AlertDialog show() {
            final AlertDialog dialog = create();
            dialog.show();
            return dialog;
        }
    }
}
```

## 7 总结

Builder模式是属于设计模式中创建模式的一种，通常作为配置类的构建器将配置的构建和表示分离。Builder模式比较常见的实现形式是通过调用链实现，这样使得代码更简洁、更便于扩展。