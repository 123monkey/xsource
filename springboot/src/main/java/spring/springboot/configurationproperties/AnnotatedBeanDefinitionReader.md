AnnotatedBeanDefinitionReader
##1.需求
为注解类生成对应BeanDefinition
##2.方案
1.使用AnnotationConfigUtils注册BeanFactoryPostProcessor,ConfigurationClassPostProcessor在所有bean definitions创建但未实例化时对这些类中的注解进行加工处理

2.将class类注入到bean工厂
##3.实现
关键:AnnotationConfigUtils.registerAnnotationConfigProcessors

类:
BeanNameGenerator用于definitions创建时的命名策略
AnnotationBeanNameGenerator
