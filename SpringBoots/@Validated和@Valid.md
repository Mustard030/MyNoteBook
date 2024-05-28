 区别：
   - @Valid注解用于参数前，表示该入参需要走该类内部属性的校验      
   - @Validated注解用在类上，本质上是一个切面注解，表示这个注解上的类需要被切面代理    

在Controller的入参里直接使用@Valid就行，会自动走校验，但是在Service上就算加了@Valid也不会走校验，需要在Service类上添加@Validated才会走校验
