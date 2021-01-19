---
title: GraphQl-Java(一)Schema
date: 2021-01-19 10:13:53
tags: GraphQl
description: 本文基于GraphQl-java:v1.6版本官方文档中文翻译。
---

## 引入依赖

[版本列表](https://github.com/graphql-java/graphql-java/releases)

### Gradle

```grovvoy
repositories {
    mavenCentral()
}

dependencies {
    compile 'com.graphql-java:graphql-java:15.0'
}
```

### Maven

```xml
<dependency>
    <groupId>com.graphql-java</groupId>
    <artifactId>graphql-java</artifactId>
    <version>15.0</version>
</dependency>
```

## Schema——大纲

创建一个Schema有两种方式：

1. 推荐SDL(special graphql dsl):
    ```sdl
    type Foo {
        bar: String
    }
    ```

2. java code:
    ```java
    GraphQLObjectType fooType = newObject()
        .name("Foo")
        .field(newFieldDefinition()
                .name("bar")
                .type(GraphQLString))
        .build();
    ```

## DataFetcher and TypeResolver

`DataFetcher`用来向field提供数据，每一个`field`都包含一个DataFetcher,默认采用`PropertyDataFetcher`。

`PropertyDataFetcher` 从`Map`或者`Java Bean`中获取数据，所以当field name与map的key，或者类的元素名称的相匹配，就不需要定义`DataFetcher`了。

`TypeResolver`帮助`graphql-java`判断值的类型。

想象一下，你有一个接口MagicUserType，用来解析一系列java类Wizard,Witch和Necromancer。类型检查器负责检查一个运行时对象，并且判断用哪个`GraphqlObjectType`去响应这个对象，对应的应该用哪个`data fetchers`和field去运行。

```java
new TypeResolver() {
    @Override
    public GraphQLObjectType getType(TypeResolutionEnvironment env) {
        Object javaObject = env.getObject();
        if (javaObject instanceof Wizard) {
            return env.getSchema().getObjectType("WizardType");
        } else if (javaObject instanceof Witch) {
            return env.getSchema().getObjectType("WitchType");
        } else {
            return env.getSchema().getObjectType("NecromancerType");
        }
    }
};
```

## 用SDL创建一个Schema

当通过schema创建一个SDL，你应该定义好`DataFetcher`和`TypeResolver`。

比如创建一个starWarsSchema.graphqls:

```sdl
schema {
    query: QueryType
}

type QueryType {
    hero(episode: Episode): Character
    human(id : String) : Human
    droid(id: ID!): Droid
}


enum Episode {
    NEWHOPE
    EMPIRE
    JEDI
}

interface Character {
    id: ID!
    name: String!
    friends: [Character]
    appearsIn: [Episode]!
}

type Human implements Character {
    id: ID!
    name: String!
    friends: [Character]
    appearsIn: [Episode]!
    homePlanet: String
}

type Droid implements Character {
    id: ID!
    name: String!
    friends: [Character]
    appearsIn: [Episode]!
    primaryFunction: String
}
```

这个静态文件`starWarsSchema.graphqls`包含了field和type的定义，但是你还需要一个`runtimeWiring（运行时注入）`来把这个静态文件转换为一个可运行的schema。

`runtimeWiring`必须包含`DataFetcher`、`TypeResolver`和自定义的`Scalar`。

你可以用下面的建造者模式去创建报文：

```java
 RuntimeWiring buildRuntimeWiring() {
    return RuntimeWiring.newRuntimeWiring()
            .scalar(CustomScalar)
            // this uses builder function lambda syntax
            .type("QueryType", typeWiring -> typeWiring
                    .dataFetcher("hero", new StaticDataFetcher(StarWarsData.getArtoo()))
                    .dataFetcher("human", StarWarsData.getHumanDataFetcher())
                    .dataFetcher("droid", StarWarsData.getDroidDataFetcher())
            )
            .type("Human", typeWiring -> typeWiring
                    .dataFetcher("friends", StarWarsData.getFriendsDataFetcher())
            )
            // you can use builder syntax if you don't like the lambda syntax
            .type("Droid", typeWiring -> typeWiring
                    .dataFetcher("friends", StarWarsData.getFriendsDataFetcher())
            )
            // or full builder syntax if that takes your fancy
            .type(
                    newTypeWiring("Character")
                            .typeResolver(StarWarsData.getCharacterTypeResolver())
                            .build()
            )
            .build();
}
```

最终，你可以把静态的大纲和注入结合到一起来创建一个可运行的schema：

```java
SchemaParser schemaParser = new SchemaParser();
SchemaGenerator schemaGenerator = new SchemaGenerator();

File schemaFile = loadSchema("starWarsSchema.graphqls");

TypeDefinitionRegistry typeRegistry = schemaParser.parse(schemaFile);
RuntimeWiring wiring = buildRuntimeWiring();
GraphQLSchema graphQLSchema = schemaGenerator.makeExecutableSchema(typeRegistry, wiring);
```

另外在建造者模式的基础上，`TypeResolver`和`DataFetcher`可以使用`WiringFactory`接口来注入。这会允许更多的动态运行时，通过schema的定义去判断应该注入什么东西。你可以通过阅读SDL来判断生成哪种runtime。

```java
RuntimeWiring buildDynamicRuntimeWiring() {
    WiringFactory dynamicWiringFactory = new WiringFactory() {
        @Override
        public boolean providesTypeResolver(TypeDefinitionRegistry registry, InterfaceTypeDefinition definition) {
            return getDirective(definition,"specialMarker") != null;
        }

        @Override
        public boolean providesTypeResolver(TypeDefinitionRegistry registry, UnionTypeDefinition definition) {
            return getDirective(definition,"specialMarker") != null;
        }

        @Override
        public TypeResolver getTypeResolver(TypeDefinitionRegistry registry, InterfaceTypeDefinition definition) {
            Directive directive  = getDirective(definition,"specialMarker");
            return createTypeResolver(definition,directive);
        }

        @Override
        public TypeResolver getTypeResolver(TypeDefinitionRegistry registry, UnionTypeDefinition definition) {
            Directive directive  = getDirective(definition,"specialMarker");
            return createTypeResolver(definition,directive);
        }

        @Override
        public boolean providesDataFetcher(TypeDefinitionRegistry registry, FieldDefinition definition) {
            return getDirective(definition,"dataFetcher") != null;
        }

        @Override
        public DataFetcher getDataFetcher(TypeDefinitionRegistry registry, FieldDefinition definition) {
            Directive directive = getDirective(definition, "dataFetcher");
            return createDataFetcher(definition,directive);
        }
    };
    return RuntimeWiring.newRuntimeWiring()
            .wiringFactory(dynamicWiringFactory).build();
}
```

## 创建一个schema定义

当schema生成时，`DataFetcher`和`TypeResolver`可以在类型创建时提供：

来活了，未完待续。。。