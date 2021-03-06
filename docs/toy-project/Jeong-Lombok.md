---
layout: default
title: Jeong-Lombok(진행 중)
parent: Toy-Projects
nav_order: 1
---
## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

# ✋**Java11 com.sun import 문제**
- [JDK 9 이상 내부 API의 javac를 사용하지 못하는 이유](https://jeongcode.github.io/docs/exception/Java/#jdk-11-comsun-import-%EB%AC%B8%EC%A0%9C)
- `com.sun.*` JDK 9 이상 부터 모듈 기능이 추가 되면서 내부 라이브러리를 보호하게 되었다고한다. 메이븐 추가 설정 시 내부 라이브러리를 사용할 수 있지만 권장하지는 않는다고 한다.
- 이러한 이유로 외부 라이브러리 [javac.jar](https://jar-download.com/artifacts/org.kohsuke.sorcerer/sorcerer-javac/0.11/source-code)를 추가하여 아래의 방법을 사용해보았지만
- `package com.sun.tools.javac.util is declared in module jdk.compiler, which does not export it to the unnamed module` 예외는 계속 발생하였다.

## 외부 라이브러리 등록
1. Project Structure - (Shift + Ctrl + Alt + S)
- ![](../../assets/images/toy-project/1.png)
1. 원하는 .jar 선택
- ![](../../assets/images/toy-project/2.png)
1. 프로젝트 선택
- ![](../../assets/images/toy-project/3.png)

> 🚨 **Global Libraries설정**
> - ![](../../assets/images/toy-project/4.png)

## maven 의존성 추가
```html
<dependency>
  <groupId>org.kohsuke.sorcerer</groupId>
  <artifactId>sorcerer-javac</artifactId>
  <version>0.11</version>
</dependency>
```

## 정리
- <span style="color:red; font-weight:bold">JDK 8 버전으로 진행</span>

> Java 9에서 소개 된 Jigsaw 이전에 maven은 javac클래스를 참조하는 프로젝트를 빌드하기 위해 컴파일 타임의 클래스 경로에 jar를 전달해야합니다
> **Maven jdk.tools 래퍼 의존성 추가 [출처](https://github.com/olivergondza/maven-jdk-tools-wrapper)**
> ```html
> <dependency>
> <groupId>com.github.olivergondza</groupId>
>  <artifactId>maven-jdk-tools-wrapper</artifactId>
> <version>0.1</version>
> </dependency>
> ```

# **Jeong-Lombok** [Github](https://github.com/jeongcode/jeong-lombok)
- `@JeongGetter`
  - 롬복은 내부 컴파일러 `com.sun.*`를 사용하여 추상구문트리를 직접 수정한다.
- `@JeongPoetGetter`
  - **[JavaPoet](https://www.baeldung.com/java-poet)** 을 사용하여 클래스에서만 사용 가능한 어노테이션
- `@JeongEntity`
  - 위의 `@JeongGetter`를 활용하여 롬복의 기능을 만들어보자
- **JavaPoet , AnnotationProcessor , Reflection , 자바컴파일러** 의 이해를 위한 프로젝트이다.
- 먼저 `@JeongGetter`를 통해 AST를 직접 조작하는 코드를 보자
- JDK 8 , IntelliJ 2020.2.4

📌 **[Lombok은 어떻게 동작하는걸까? (AnnotationProcessor에 대해)](https://jeongcode.github.io/docs/java/Annotation%20Processor/)**
{: .fh-default .fs-4 }

📌 **[Java 컴파일러](https://jeongcode.github.io/docs/java/javac-principle/)**
{: .fh-default .fs-4 }

## **`com.sun.*` AST 예제**

### **`@JeongGetter`**

📌 **[Getter 참고](https://catch-me-java.tistory.com/49)**
{: .fh-default .fs-4 }
- [juejin.cn](https://juejin.cn/post/6844904082084233223#heading-1)

- com.sun.* 사용

```java
@AutoService(Processor.class)
@SupportedAnnotationTypes("org.example.JeongGetter")
@SupportedSourceVersion(SourceVersion.RELEASE_8)
public class GetterProcessor extends AbstractProcessor {

    private Messager messager;
    private ProcessingEnvironment processingEnvironment;
    private Trees trees;
    private TreeMaker treeMaker;
    private Names names;
    private Context context;

    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
        JavacProcessingEnvironment javacProcessingEnvironment = (JavacProcessingEnvironment) processingEnv;
        this.processingEnvironment = processingEnv;
        this.messager = processingEnv.getMessager();
        this.trees = Trees.instance(processingEnv);
        this.context = javacProcessingEnvironment.getContext();
        this.treeMaker = TreeMaker.instance(context);
        this.names = Names.instance(context);
    }

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        System.out.println("call process =" + annotations);
        TreePathScanner<Object, CompilationUnitTree> scanner = new TreePathScanner<Object, CompilationUnitTree>(){
            @Override
            public Trees visitClass(ClassTree classTree, CompilationUnitTree unitTree){
                JCTree.JCCompilationUnit compilationUnit = (JCTree.JCCompilationUnit) unitTree;
                if (compilationUnit.sourcefile.getKind() == JavaFileObject.Kind.SOURCE){
                    compilationUnit.accept(new TreeTranslator() {
                        @Override
                        public void visitClassDef(JCTree.JCClassDecl jcClassDecl) {
                            super.visitClassDef(jcClassDecl);
                            List<JCTree> fields = jcClassDecl.getMembers();
                            System.out.println("class name = " + jcClassDecl.getSimpleName());
                            for(JCTree field : fields){
                                System.out.println("class field = " + field);
                                if (field instanceof JCTree.JCVariableDecl){
                                    List<JCTree.JCMethodDecl> getters = createGetter((JCTree.JCVariableDecl) field);
                                    for(JCTree.JCMethodDecl getter : getters){
                                        jcClassDecl.defs = jcClassDecl.defs.prepend(getter);
                                    }
                                }
                            }
                        }
                    });
                }
                return trees;
            }
        };

        for (final Element element : roundEnv.getElementsAnnotatedWith(JeongGetter.class)) {
            System.out.println("call process - getPath() element = " + element);
            if(element.getKind() != ElementKind.CLASS){
                processingEnv.getMessager().printMessage(Diagnostic.Kind.ERROR, "@JeongGetter annotation cant be used on" + element.getSimpleName());
            }else{
                processingEnv.getMessager().printMessage(Diagnostic.Kind.NOTE, "@JeongGetter annotation Processing " + element.getSimpleName());
                final TreePath path = trees.getPath(element);
                scanner.scan(path, path.getCompilationUnit());
            }
        }
        return true;
    }

    public List<JCTree.JCMethodDecl> createGetter(JCTree.JCVariableDecl var){
        String str = var.name.toString();
        String upperVar = str.substring(0,1).toUpperCase()+str.substring(1,var.name.length());
        System.out.println(str + " Create Getter");
        return List.of(
                // treeMaker.Modifiers -> syntax tree node 에 접근하여 수정 및 삽입하는 역할
                treeMaker.MethodDef(
                        treeMaker.Modifiers(1), // public
                        names.fromString("get".concat(upperVar)), // 메서드 명
                        (JCTree.JCExpression) var.getType(), // return type
                        List.nil(),
                        List.nil(),
                        List.nil(),
                        treeMaker.Block(1, List.of(treeMaker.Return((treeMaker.Ident(var.getName()))))),
                        null));
    }
}
```

```
call process =[org.example.JeongGetter]
call process - getPath() element = org.example.TestEntity
class name = TestEntity
class field =
public <init>() {
    super();
}
class field = private String TestEntity_first
TestEntity_first Create Getter
class field = private String TestEntity_second
TestEntity_second Create Getter
class field = private String TestEntity_third
TestEntity_third Create Getter
call process - getPath() element = org.example.TestEntity2
class name = TestEntity2
class field =
public <init>() {
    super();
}
class field = private String TestEntity2_first
TestEntity2_first Create Getter
call process =[]
```
- ![](../../assets/images/toy-project/5.png)
- 해당 target의 디컴파일된 코드를 보면 `@Retention`에 의해 `@JeongGetter`는 포함되어 있지 않고 필드들의 getter메소드를 볼 수 있다.
- 공개된 API가 아닌 컴파일러의 내부 클래스를 사용하여 AST를 수정한 것이다.
- 특히 이클립스의 경우엔 java agent를 사용하여 컴파일러 클래스까지 조작하여 사용한다. 해당 클래스들 역시 공개된 API가 아니다보니 버전 호환성에 문제가 생길 수 있고 언제라도 그런 문제가 발생해도 이상하지 않다.
- **JavaPoet으로는 클래스를 생성하고 코드를 생성할 순 있지만 , 컴파일 시점 특정 코드를 수정하고 삽입할 수는 없다.**


***

## **JavaPoet 예제**
```html
<dependency>
   <groupId>com.squareup</groupId>
   <artifactId>javapoet</artifactId>
   <version>1.11.1</version>
</dependency>
```

### **`@JeongPoetGetter`**
- **클래스에만 허용 하며 해당 엔티티를 상속받는다.**
- 클래스가 아닌 곳에 작성 시 컴파일 에러
- ![](../../assets/images/toy-project/6.png)
- 제네릭 포함 시 `return` 클래스 에러
```java
protected List<String> tStringList;
protected List<Integer> tIntList;
protected Map<String,String> tMap;
```
- ![](../../assets/images/toy-project/7.png)

```java
@AutoService(Processor.class)
@SupportedAnnotationTypes("org.example.annotation.JeongPoetGetter")
@SupportedSourceVersion(SourceVersion.RELEASE_8)
public class PoetGetterProcessor extends AbstractProcessor {

    // ture를 리턴 시 이 어노테이션의 타입을 처리하였으니 (여러 라운드의) 다음 프로세서들은 이 어노테이션의 타입을 다시 처리하진 않는다.
    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        Set<? extends Element> elements = roundEnv.getElementsAnnotatedWith(JeongPoetGetter.class);

        // @JeongPoetGetter 어노테이션이 적용된 클래스,인터페이스.. 가져오기
        for(Element element : elements){
            Name elementName = element.getSimpleName();
            if(element.getKind() != ElementKind.CLASS){
                // JeongPoetGetter 어노테이션을 클래스가 아닌 다른 곳에 작성하였다면
                // 컴파일 에러 처리
                processingEnv.getMessager().printMessage(Diagnostic.Kind.ERROR , "@JeongPoetGetter annotation cant be used on" + element.getSimpleName());
            }
            else{
                // 인터페이스에 제대로 작성하였다면
                // 로깅
                processingEnv.getMessager().printMessage(Diagnostic.Kind.NOTE , "@JeongPoetGetter annotation Processing " + element.getSimpleName());
            }

            // javax.lang.model.element public interface TypeElement를
            // com.squareup.javapoet public final class ClassName 으로 변환할 수 있다.
            TypeElement typeElement = (TypeElement) element;
            // ClassName으로 그 클래스에 대한 정보를 사용할 수 있다.
            ClassName className = ClassName.get(typeElement);

            // 해당 클래스의 필드 가져오기
            // javax.lang.model.element (https://docs.oracle.com/javase/8/docs/api/javax/lang/model/element/TypeElement.html)
            List<MethodSpec> methodSpecArr = new ArrayList<MethodSpec>();
            for(Element field : element.getEnclosedElements()){
                if(field.getKind() == ElementKind.FIELD){
                    String fieldName = field.getSimpleName().toString();
                    String upperName = fieldName.substring(0,1).toUpperCase() + fieldName.substring(1 , fieldName.length());
                    // com.squareup.javapoet public final class MethodSpec을 사용하여 메서드를 만들 수 있다.
                    System.out.println(field.asType());
                    try {
                        MethodSpec getter = MethodSpec.methodBuilder("get" + upperName)
                                .addModifiers(Modifier.PUBLIC)
                                // 참조형 타입일 시 Class.forName , 프리미티브 타입일 시 getType 호출
                                .returns(("DECLARED".equals(field.asType().getKind().toString())) ? Class.forName(field.asType().toString()) : getType(field.asType().getKind()))
                                .addStatement("return super." + fieldName)
                                .build();
                        methodSpecArr.add(getter);
                    } catch (ClassNotFoundException e) {
                        processingEnv.getMessager().printMessage(Diagnostic.Kind.ERROR , "ClassNotFoundException : " + e);
                    }
                }
            }

            // com.squareup.javapoet.TypeSpec을 사용하여 클래스를 만들 수 있다.
            // classBuilder의 addMethods(Iterable<MethodSpec> methodSpecs)
            TypeSpec makeClass = TypeSpec.classBuilder(elementName + "Entity")
                    .addModifiers(Modifier.PUBLIC)
                    .superclass(className)
                    .addMethods(methodSpecArr)
                    .build();

            // 위에서 정의한 Spec을 사용하여 실제 Source 코드에 삽입해보자

            // 1. javax.annotation.processing public interface Filer 인터페이스를 가져오자
            Filer filer = processingEnv.getFiler();

            // 2. com.squareup.javapoet public final class JavaFile 을 사용
            // 2.1 makeClass를 해당 패키지에 만들어 달라.
            // 2.2 위에서 가저온 filer를 사용하여 써달라.
            try {
                JavaFile.builder(className.packageName() , makeClass)
                        .build()
                        .writeTo(filer);
            } catch (Exception e) {
                processingEnv.getMessager().printMessage(Diagnostic.Kind.ERROR , "FATA ERROR : " + e);
            }
        }
        return true;
    }

    // 프리미티브 타입일 때
    private Class getType(TypeKind kind){
        switch(kind) {
            case BOOLEAN:
                return boolean.class;
            case BYTE:
                return byte.class;
            case SHORT:
                return short.class;
            case INT:
                return int.class;
            case LONG:
                return long.class;
            case CHAR:
                return char.class;
            case FLOAT:
                return float.class;
            case DOUBLE:
                return double.class;
            default:
                throw new AssertionError();
        }
    }
}

```

![](../../assets/images/toy-project/8.png)
- `@JeongPoetGetter` 어노테이션이 작성된 클래스를 상속 받는다.

***

## **`@JeongEntity`**

- `com.sun.*` JDK 자바컴파일러 내부 클래스를 사용하여 진행한다.
- 클래스에만 작성 가능 하다.
- **기능**
  - Getter
  - Setter
  - toString
  - equals
  - Constructor
