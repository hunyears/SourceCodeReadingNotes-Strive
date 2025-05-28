一：简述：
    
    简述：此类是：openFeign入口类，主要职责为：
    扫描服务内包含了 FeignClient 注解的类，将其注入到 spring容器内
    
    具体步骤如下：
        1.1：读取 @EnableFeignClients 注解配置
        1.2：注册默认配置（如 defaultConfiguration）
        1.3：确定扫描包路径或指定接口类
        1.4：扫描所有 @FeignClient 接口
        1.5：校验并读取注解元信息
        1.6：注册每个接口的配置信息
        1.7：为每个接口注册一个 FeignClientFactoryBean 到容器
    
二：类定义：

    class FeignClientsRegistrar
    		implements ImportBeanDefinitionRegistrar, ResourceLoaderAware, EnvironmentAware {
    }
    
三：核心方法：
    
    // openFeign：入口
    @Override
    public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
        // 注册默认配置（如 defaultConfiguration）
        registerDefaultConfiguration(metadata, registry);
        // 扫描并注册所有 @FeignClient 接口
        registerFeignClients(metadata, registry);
    }
    
    // 注册默认配置
    private void registerDefaultConfiguration(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
        // 获取 @EnableFeignClients 注解中的所有属性
        Map<String, Object> defaultAttrs = metadata
                .getAnnotationAttributes(EnableFeignClients.class.getName(), true);
    
        if (defaultAttrs != null && defaultAttrs.containsKey("defaultConfiguration")) {
            // 生成配置名称，支持内部类
            String name;
            if (metadata.hasEnclosingClass()) {
                name = "default." + metadata.getEnclosingClassName();
            } else {
                name = "default." + metadata.getClassName();
            }
    
            // 注册 Feign 的默认配置
            registerClientConfiguration(registry, name, defaultAttrs.get("defaultConfiguration"));
        }
    }
    
    // 注册 Feign 客户端配置类
    private void registerClientConfiguration(BeanDefinitionRegistry registry, Object name,
            Object configuration) {
        // 创建 FeignClientSpecification 配置 Bean 的定义
        BeanDefinitionBuilder builder = BeanDefinitionBuilder
                .genericBeanDefinition(FeignClientSpecification.class);
    
        // 构造函数注入：name + 配置类
        builder.addConstructorArgValue(name);
        builder.addConstructorArgValue(configuration);
    
        // 注册为 Bean，Bean 名：default.xx.FeignClientSpecification
        registry.registerBeanDefinition(
                name + "." + FeignClientSpecification.class.getSimpleName(),
                builder.getBeanDefinition());
    }
    
    // 注册 Feign 接口
    public void registerFeignClients(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
        // 创建一个扫描器（ClassPathBeanDefinitionScanner 的变种）
        ClassPathScanningCandidateComponentProvider scanner = getScanner();
        scanner.setResourceLoader(this.resourceLoader);
    
        Set<String> basePackages;
    
        // 读取 @EnableFeignClients 注解的属性
        Map<String, Object> attrs = metadata
                .getAnnotationAttributes(EnableFeignClients.class.getName());
    
        // 创建一个只扫描带有 @FeignClient 注解的过滤器
        AnnotationTypeFilter annotationTypeFilter = new AnnotationTypeFilter(FeignClient.class);
    
        // 获取 clients 属性（可指定要扫描的接口）
        final Class<?>[] clients = attrs == null ? null : (Class<?>[]) attrs.get("clients");
    
        if (clients == null || clients.length == 0) {
            // 如果未显式指定接口类，则扫描 basePackages 中的所有类，查找带 @FeignClient 注解的接口
            scanner.addIncludeFilter(annotationTypeFilter);
            basePackages = getBasePackages(metadata);
        } else {
            // 指定了具体的接口类，通过 package 名确定扫描路径
            final Set<String> clientClasses = new HashSet<>();
            basePackages = new HashSet<>();
            for (Class<?> clazz : clients) {
                basePackages.add(ClassUtils.getPackageName(clazz));
                clientClasses.add(clazz.getCanonicalName());
            }
    
            // 只匹配指定的类名和 @FeignClient 注解
            AbstractClassTestingTypeFilter filter = new AbstractClassTestingTypeFilter() {
                @Override
                protected boolean match(ClassMetadata metadata) {
                    String cleaned = metadata.getClassName().replaceAll("\\$", ".");
                    return clientClasses.contains(cleaned);
                }
            };
            scanner.addIncludeFilter(
                    new AllTypeFilter(Arrays.asList(filter, annotationTypeFilter)));
        }
    
        // 开始扫描指定包路径
        for (String basePackage : basePackages) {
            Set<BeanDefinition> candidateComponents = scanner.findCandidateComponents(basePackage);
            for (BeanDefinition candidateComponent : candidateComponents) {
                if (candidateComponent instanceof AnnotatedBeanDefinition) {
                    AnnotatedBeanDefinition beanDefinition = (AnnotatedBeanDefinition) candidateComponent;
                    AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();
    
                    // 校验是否为接口
                    Assert.isTrue(annotationMetadata.isInterface(),
                            "@FeignClient can only be specified on an interface");
    
                    // 获取 @FeignClient 注解的属性
                    Map<String, Object> attributes = annotationMetadata
                            .getAnnotationAttributes(FeignClient.class.getCanonicalName());
    
                    // 获取客户端名称
                    String name = getClientName(attributes);
    
                    // 注册每个 Feign 接口的配置类
                    registerClientConfiguration(registry, name, attributes.get("configuration"));
    
                    // 注册该 Feign 接口为一个 Bean
                    registerFeignClient(registry, annotationMetadata, attributes);
                }
            }
        }
    }
    
    // 注册具体 Feign 接口 Bean
    private void registerFeignClient(BeanDefinitionRegistry registry,
            AnnotationMetadata annotationMetadata, Map<String, Object> attributes) {
        String className = annotationMetadata.getClassName();
    
        // 创建一个 FeignClientFactoryBean 的 Bean 定义
        BeanDefinitionBuilder definition = BeanDefinitionBuilder
                .genericBeanDefinition(FeignClientFactoryBean.class);
    
        // 属性校验（可选）
        validate(attributes);
    
        // 填充属性
        definition.addPropertyValue("url", getUrl(attributes));
        definition.addPropertyValue("path", getPath(attributes));
        String name = getName(attributes);
        definition.addPropertyValue("name", name);
        String contextId = getContextId(attributes);
        definition.addPropertyValue("contextId", contextId);
        definition.addPropertyValue("type", className);
        definition.addPropertyValue("decode404", attributes.get("decode404"));
        definition.addPropertyValue("fallback", attributes.get("fallback"));
        definition.addPropertyValue("fallbackFactory", attributes.get("fallbackFactory"));
    
        // 设置自动注入模式
        definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
    
        // 定义别名
        String alias = contextId + "FeignClient";
        AbstractBeanDefinition beanDefinition = definition.getBeanDefinition();
    
        // 设置 primary 属性
        boolean primary = (Boolean) attributes.get("primary");
        beanDefinition.setPrimary(primary);
    
        // 设置 qualifier（如果有）
        String qualifier = getQualifier(attributes);
        if (StringUtils.hasText(qualifier)) {
            alias = qualifier;
        }
    
        // 注册 Bean 到 Spring 容器
        BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className,
                new String[] { alias });
        BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
    }





