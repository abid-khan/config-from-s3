config-from-s3
==============

In this blog I will show you how we can  load basic configuration properties for spring application from S3.
Traditionally what we , we keep configuration file (application.properties) in class path and configure spring to load it during context loading.

```xml
<bean id="propertyConfigurer"
		class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer "
		depends-on="s3Service">
		<property name="systemPropertiesModeName" value="SYSTEM_PROPERTIES_MODE_OVERRIDE" />
		<property name="locations">
			<list>
				<value>classpath:application.properties</value>
			</list>
		</property>

	</bean>
```


Now assume your application is running in cluster with multiple instances and you want to keep configuration file at some centralized location. You have to mounta common node or directory for the same. Here I will show  you how we can use AWS  S3 for this purpose.


Prerequisite
==============
You must have valid AWS access and secret  key along woth a valid bucket.You can set these value as environment values and fetch these whenever required. 

In this exmaple I have passed these values in plain text. You can modify as per your need.


First we need to define a bean for S3 authentication. And this can be done as below. You need to update valied access key and secret key.

```java
@Bean
	@Qualifier("amazonS3")
	public AmazonS3 amazonS3() {
		AmazonS3 amazonS3 = new AmazonS3Client(new BasicAWSCredentials(
				"access key", "secret key"));

		return amazonS3;

	}
```

We have to define one more bean for properties as a replacement for above XML configuraiton. Here instead of PropertyPlaceholderConfigurer, I have used custom implementation of it in which properties from S3 is loaded.
Path is the directory(excluding bucket name) of S3 in which application.properties resides.

```java
@Bean
	@DependsOn("amazonS3")
	public org.springframework.beans.factory.config.PropertyPlaceholderConfigurer properties() {

		org.springframework.beans.factory.config.PropertyPlaceholderConfigurer configurer = new S3PropertyPlaceholderConfigurer(
				amazonS3(), "path", "application.properties");

		configurer.setIgnoreResourceNotFound(true);
		configurer
				.setSystemPropertiesModeName("SYSTEM_PROPERTIES_MODE_OVERRIDE");

		return configurer;
	}
```
And custom implementation of PropertyPlaceholderConfigurer.
You need use vaid bucket name.

```java
	
	public class S3PropertyPlaceholderConfigurer extends
		PropertyPlaceholderConfigurer {

	private AmazonS3 amazonS3;

	private String S3_RESOURCE_PATH;
	private String S3_RESOURCE_NAME;

	public S3PropertyPlaceholderConfigurer(AmazonS3 amazonS3, String resourcePath,
			String resourceName) {
		this.amazonS3 = amazonS3;
		this.S3_RESOURCE_PATH = resourcePath;
		this.S3_RESOURCE_NAME = resourceName;
		
	}

	@Override
	public void postProcessBeanFactory(ConfigurableListableBeanFactory arg0)
			throws BeansException {
		loadResources();
		super.postProcessBeanFactory(arg0);
	}
	
	

	@Override
	public void setIgnoreResourceNotFound(boolean ignoreResourceNotFound) {
		super.setIgnoreResourceNotFound(ignoreResourceNotFound);
	}

	private void loadResources() {

		List<Resource> targetResources = new ArrayList<Resource>();
		try {

			S3Object obj = amazonS3.getObject(new GetObjectRequest(
					"bucket name", S3_RESOURCE_PATH + "/" +  S3_RESOURCE_NAME));
			
			S3File s3File = new S3File(new ByteArrayInputStream(
					IOUtils.toByteArray(obj.getObjectContent())), obj
					.getObjectMetadata().getContentLength(), S3_RESOURCE_NAME);
			s3File.getInputStream().close();
			
			
			
			targetResources.add(new ByteArrayResource(IOUtils.toByteArray(s3File
					.getInputStream())));

		} catch (IOException ex) {
			ex.printStackTrace();
		}

		super.setLocations(targetResources.toArray(new Resource[targetResources
				.size()]));
	}

}

```

