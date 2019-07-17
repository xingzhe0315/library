###基于Gradle Transform、ASM编译期修改字节码文件实现无侵入自动插桩

在日常编写代码时，遇到异常我们通常是用try catch将异常捕获然后做相应的异常处理，但是现在我们想要知道我们的应用程序在运行过程中都会遇到哪些异常以及是否频繁，那么最笨的方法就是在没个catch块中调用一个记录异常的方法，如果工程比较大的话，这也是个非常巨大的工作量，而且以后每次try一个异常都要加一个方法调用，很不友好，于是我们就想到通过自定义gradle插件在编译期修改字节码来实现无侵入式自动插桩。

关于编译期修改字节码，配合Gradle插件使用Transom是通常会采用的方法，字节码工具主流的有Javassist与ASM:

**ASM** ：直接操作字节码指令，执行效率高，要是使用者掌握Java类字节码文件格式及指令，对使用者的要求比较高。

**Javassist** 提供了更高级的API，执行效率相对较差，但无需掌握字节码指令的知识，对使用者要求较低。

出于效率考虑，我们选择ASM来作为操作class的工具。

![Android编译流程](https://developer.android.google.cn/images/tools/studio/build-process_2x.png)

上图是Android大致的编译流程，transform可以在编译成class之后打包为dex之前修改class实现编译期修改字节码，实现无侵入式插桩。

实现的大致流程：

1. 自定义gradle插件，实现org.gradle.api.Plugin接口，在apply方法中注册自定义的Transform。
2. 自定义Transform，继承com.android.build.api.transform.Transform，实现相关方法，重写transform方法实现修改class的核心逻辑。
3. 实现ClassVisitor，并将一个ClassWriter对象传入，在visitMethod方法中对我们需要插桩方法进行修改。
4. 通过ClassReader实现对类的读取。
5. 返回ClassWriter.toByteArray()，这就是修改后的字节码将其写入目标文件实现对class的修改。

下面简单介绍一下我们用到的工具。

#### Transform

[Gradle Tramsform](http://google.github.io/android-gradle-dsl/javadoc/2.1/com/android/build/api/transform/Transform.html)是谷歌官方提供给开发者在项目构建阶段由class到dex转换期间修改class文件的一套api。

每添加一个transform，就会创建一个新的任务（task），transform之间的依赖关系在添加时指定，所有transform是一个链式的调用，前一个transform的输出结果作为下一个transform的输入内容。

~~~java
public void registerTransform(Transform transform, Object... dependencies) {
	  this.transforms.add(transform);
    this.transformDependencies.add(Arrays.asList(dependencies));
}
~~~

上面就是添加transform的方法，添加时可以通过后面的可变参数指定它的依赖关系。通常情况下需要在自定义的Plugin的apply方法中注册transform。

Transform类有几个抽象方法需要实现：

```java
/**
*返回transform的名称
*/
public abstract  String  getName（）
/**
*返回transform可以消费的数据的类型，可能不止一种，但是必须是 QualifiedContent.DefaultContentType类型，
*QualifiedContent.DefaultContentType只有CLASSES和RESOURCES两种
*/
public abstract Set<QualifiedContent.ContentType> getInputTypes()
/**
*返回transform的的范围
*PROJECT(1),SUB_PROJECTS(4),EXTERNAL_LIBRARIES(16),TESTED_CODE(32),PROVIDED_ONLY(64),
*/
public abstract Set<QualifiedContent.Scope> getScopes()
/**
*返回Transform是否可以执行增量工作。
*如果是，则TransformInput可以包含已更改/删除/添加的文件的列表，除非其他内容触发非增量运行。
*/
public abstract boolean isIncremental()
```

核心逻辑，需要在transform方法中实现

```java
void transform(TransformInvocation transformInvocation) throws TransformException, InterruptedException, IOException
```

它接受一个TransformInvocation类型的参数，可以调用transformInvocation.inputs来获取TransformInput的集合，TransformInput由DirectoryInput和JarInput两种类型的集合组成，两者都包含了Scope、File和ContentType等信息。通过DirectoryInput或JarInput对class文件处理后，如果需要将改动输出，通过TransformInvocation 的TransformOutputProvider来拿到输出目录，然后在将DirectoryInput或JarInput的改变过后的文件拷贝到该目录

```groovy
@Override
void transform(TransformInvocation transformInvocation) throws TransformException, InterruptedException, IOException {
    super.transform(transformInvocation)
    def inputs = transformInvocation.inputs
    outputProvider = transformInvocation.outputProvider
    for (TransformInput transformInput : inputs) {
        for (DirectoryInput directoryInput : transformInput.directoryInputs) {
            File root = directoryInput.file
          	//do something
            def dest = outputProvider.getContentLocation(
                    root.getAbsolutePath(),
                    directoryInput.contentTypes,
                    directoryInput.scopes,
                    Format.DIRECTORY
            )
            FileUtils.copyDirectory(root,dest)
        }

        for (JarInput jarInput : transformInput.jarInputs) {
            //do something
            def dest = outputProvider.getContentLocation(destName + "_" + hex, jarInput.contentTypes, jarInput.scopes, Format.JAR)
            FileUtils.copyFile(jarInput.file,dest)
        }
    }
}
```

接下来可以通过遍历并过滤文件名来拿到class文件，并对class文件做处理

####ASM

[ASM](https://asm.ow2.io/)是一个Java字节码的分析，操作工具，它一般用于修改已经存在的class或者动态生成class。它有几个比较常用的类，ClassVisitor，ClassWriter，ClassReader，

ClassVisitor: 一个用来访问class文件的访问者，他的方法必须按照下面的顺序被调用visit() —>visitSource() ->visitModule() —>visitNestHost()—>visitOuterClass()—>visitAnnotation()—>visitTypeAnnotation()—>visitAttribute()—>visitNestMember()—>visitInnerClass()—> visitField()—>visitMethod()—>visitEnd()，如果要修改方法或者在某个方法内插入代码的话可以在visitMethod方法内进行操作。

ClassWriter:派生于ClassVisitor，用来生成与Java虚拟机规范（Java Virtual Machine Specification）中的定义相匹配的类文件结构，它可以单独使用用于构建一个崭新的class，也可以与ClassReader和ClassVisitor一起来修改一个已经存在的class。

ClassReader:一个类文件的解析器，用来解析类文件结构并且调用ClassVisitor的相关方法。

```groovy
ClassWriter classWriter = new ClassWriter(ClassWriter.COMPUTE_MAXS)
ClassVisitor classVisitor = new WoundplastClassVisitor(classWriter, classNameAnalytics)
ClassReader cr = new ClassReader(srcClass)
cr.accept(classVisitor, ClassReader.EXPAND_FRAMES)
return classWriter.toByteArray()
```

上面就是相关用法，WoundplastClassVisitor是自定义的ClassVisitor，在它内部可以实现对类的修改，最后返回classWriter.toByteArray()是一个byte数组，可以将他写入目标文件，就是修改后的class。

关于方法的操作需要用到MethodVisitor，它是用来访问方法的。ClassVisitor的visitMethod方法返回一个MethodVisitor，该返回值就是对方法的操作的对象。

```groovy
@Override
MethodVisitor visitMethod(int access, String name, String desc, String signature, String[] exceptions) {

    MethodVisitor originMethodVisitor = cv.visitMethod(access, name, desc, signature, exceptions)

    MethodVisitor realMethodVisitor = new DefaultMethodVisitor(Opcodes.ASM5, originMethodVisitor) {

        private Label exceptionHandleLabel

        @Override
        void visitTryCatchBlock(Label start, Label end, Label handler, String type) {
            super.visitTryCatchBlock(start, end, handler, type)
            exceptionHandleLabel = handler
        }

        @Override
        void visitCode() {
            if ("uncaughtException".equals(name)) {
                int size = mInterfaces == null ? 0 : mInterfaces.length
                for (int i = 0; i < size; i++) {
                    String interf = mInterfaces[i]
                    if ("java/lang/Thread\$UncaughtExceptionHandler".equals(interf)) {
                        mv.visitVarInsn(Opcodes.ALOAD, 2)
                        mv.visitMethodInsn(Opcodes.INVOKESTATIC,
                                "com/common/woundplast/Woundplast",
                                "handleException",
                                "(Ljava/lang/Throwable;)V",
                                false)
                    }
                }
            }
        }

        @Override
        void visitLabel(Label label) {
            super.visitLabel(label)
            if (label == exceptionHandleLabel) {
                mv.visitInsn(Opcodes.DUP)
                mv.visitMethodInsn(Opcodes.INVOKESTATIC,
                        "com/common/woundplast/Woundplast",
                        "handleException",
                        "(Ljava/lang/Throwable;)V",
                        false)
            }
        }

    }
    return realMethodVisitor
}
```

上述代码中

```groovy
mv.visitVarInsn(Opcodes.ALOAD, 2)
mv.visitMethodInsn(Opcodes.INVOKESTATIC,
    "com/common/woundplast/Woundplast",
    "handleException",
    "(Ljava/lang/Throwable;)V",
    false)

mv.visitInsn(Opcodes.DUP)
mv.visitMethodInsn(Opcodes.INVOKESTATIC,
     "com/common/woundplast/Woundplast",
     "handleException",
     "(Ljava/lang/Throwable;)V",
     false)
```
就是在方法中插入代码的方法，ASM是通过想class插入指令来达到修改class的目的的。



####JVM相关

#####方法调用：

JVM方法调用指令有五种：

INVKLESTATIC：属于方法解析，编译时确定，适用于静态方法

INVOKESPECIAL：也属于方法解析，编译时确定，适用于构造方法<init>，私有方法，父类方法的调用。

INVOKEVIRTUAL：属于方法分派，分静态分派与动态分派

​		静态分派应用在方法重载，根据参数的静态类型确定方法调用版本。

​		动态分派应用在方法的重写，根据对象动态类型来判断调用哪个类的方法

INVOKEINTERFACE：调用接口方法，也属于方法分派

INVOKEDYNAMIC，1.7后才加入的，主要作为对其他在JVM上运行的动态语言的支持的拓展，正常情况下javac编译不会生成这个指令。

#####JVM指令：

STORE与LOAD：STORE相关指令用于将操作数占栈顶的字保存到局部变量表，LOAD相关指令用于将局部变量表中的字压入操作数栈栈顶，对于实例方法，局部变量表的第0个slot保存的是this，对于静态方法则没有。

DUP：将操作数栈栈顶复制一份并压入栈顶

在上面的应用中，第一部分就是讲局部变量表中第2个slot中的字压入栈顶，然后调用Woundplast的静态方法handleException，参数类型为Throwable，因为uncaughtException这个方法参数列表是2，而且它是一个实例方法，所以局部变量表中第0个slot是this，第二个参数Throwable要从第2个slot中取出，因此调用ALOAD_2来将它压入栈顶，第二部分是catch中的内容，进入catch时栈顶是Exception类型的值，因此我们调用DUP指令将他复制一份，然后调用Woundplast的静态方法handleException，弹出栈顶的值传入方法。



### 遇到的问题

在catch块插入代码时，栈顶元素就是Exception类型，正常情况下，如果不执行DUP指令，应该是没有问题的，但是实际上却会报错。

经过排查，不插桩的情况下javac编译生成的字节码在catch块中会调用ASTORE指令将栈顶的exception保存到局部变量表，如下图：

![image-20190712105248247](https://i.postimg.cc/fTGcKnY3/image-20190712105248247.png)

而我们插桩是插入catch块第一行的，因此如果不执行DUP指令，当我们插入的方法执行完后，操作数栈已经空了，此时在执行ASTORE就会报错，所以编译不通过，插桩后的字节码如下图：

![image-20190712105422929](https://i.postimg.cc/SxcW1TGg/image-20190712105422929.png)