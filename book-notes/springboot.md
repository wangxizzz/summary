1.Springboot会自动扫描启动类同级包及其同级包的子包所有的注解。
如果想自己控制扫描哪些包的话，使用@componentscan注解，多个包的话使用逗号分隔
如：@componentscan("com.package1,cn.package2")

2.