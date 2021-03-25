---
layout: post 
title: "JSP/Spring/Java 기반 파일 업로드 기능 구현"
date: 2021-02-27 18:55:00 +0900 
image: IE-issue-reproduction.jpg 
tags: JAVA SPRING JSP
---

## HTML - multipart/form-data file 처리
파일 업로드를 하기 위해서는 form tag, input tag 설정이 필요하다.  
아래 예시 코드를 사용하면 간단한 파일 업로드 폼을 만들수 있다.  
더 자세한 내용은 링크 걸어둔 레퍼런스를 참고하자.  

### multipart form 설정
- `method="post"`
- `enctype="multipart/form-data"`
- 자세한 설명은 [파일보내기][sile sending form data] 참고

### input 설정
- `type="file"`
- 옵션
    - `accept=".pptx"`:업로드 파일 확장자 제한
- 자세한 설명 및 가능한 옵션 확인은 [input file] 참고

{% highlight html %}
<%-- pptx 파일 1개 업로드 --%>
<form method="post" enctype="multipart/form-data" action="test/upload/requestParam" accept-charset="UTF-8">
  <input type="file" name="uploadFile" accept=".pptx">
  <input type="submit" value="requestParam upload">

<%-- pptx or png 파일 여러개 업로드 --%>
<form method="post" enctype="multipart/form-data" action="test/uploads/requestParam" accept-charset="UTF-8">
  <input multiple type="file" name="uploadFileList" accept=".pptx, .png">
  <input type="submit" value="requestParam uploads">
</form>

<%-- 이미지 파일 1개 업로드 --%>
<form method="post" enctype="multipart/form-data" action="/test/upload/commandObject" accept-charset="UTF-8">
  <input type="file" name="uploadFile" accept="image/*">
  <input type="submit" value="customModel upload">
</form>

<%-- 파워포인트 파일 여러개 업로드 --%>
<form method="post" enctype="multipart/form-data" action="/test/uploads/commandObject" accept-charset="UTF-8">
  <input multiple type="file" name="uploadFileList" accept="application/vnd.ms-powerpoint">
  <input type="submit" value="customModel uploads">
</form>
{% endhighlight %}

## Spring multipart 파일 처리

### TEST VERSION

- Spring Framework Version __5.3.5__

### multipartResolver config
`form(multipart/form-data)`에서 전달되는 파일 정보를 처리하기 위해서는 `MultipartResolver` 설정이 필요하다.   
`MultipartResolver`는 _Commons FileUpload_ 를 사용하는 `CommonsMultipartResolver`와 _Servlet 3.0_ 을 사용하는 `StandardServletMultipartResolver`가 있다.   
더 자세한 내용은 링크를 확인해보자.   
_참조_: [Spring mvc-multipart], [Spring file upload]

#### CommonsMultipartResolver config
_build.gradle_
{% highlight gradle %}
dependencies {
    implementation group: 'commons-fileupload', name: 'commons-fileupload', version: '1.4'
}
{% endhighlight %}

_WebConfig.java_
{% highlight java %}
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Bean(name = "multipartResolver")
    public CommonsMultipartResolver getCommonsMultipartResolver() {
        CommonsMultipartResolver multipartResolver = new CommonsMultipartResolver();
        multipartResolver.setMaxUploadSize(20971520);   // 20MB
        multipartResolver.setMaxInMemorySize(1048576);  // 1MB
        return multipartResolver;
    }
}
{% endhighlight %}

#### StandardServletMultipartResolver config
_WebApplicationInitializer.java_
{% highlight java %}
public class WebApplicationInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {
    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class[]{WebConfig.class};
    }
    @Override
    protected void customizeRegistration(ServletRegistration.Dynamic registration) {
        registration.setMultipartConfig(new MultipartConfigElement(""));
    }
    // ...
}
{% endhighlight %}

_WebConfig.java_
{% highlight java %}
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Bean(name = "multipartResolver")
    public StandardServletMultipartResolver getCommonsMultipartResolver() {
        return new StandardServletMultipartResolver();
    }
    // ...
}
{% endhighlight %}

### Controller
Spring에서는 `MultipartResolver`를 통해 `multipart/form-data`를 `MultipartFile`로 전달 받을 수 있다.   
`@RequestParam`을 사용하거나 `@ModelAttribute`를 사용해서 _Command Object_ 에 바인딩 할 수 있다.   
좀 더 자세한 내용은 링크를 참고하자.   
_참조_ :[Spring mvc-multipart-forms]
{% highlight java %}
/**
 * Command Object
 */
public class MultipartModel {

    private MultipartFile uploadFile;
    private List<MultipartFile> uploadFileList;
    // ...
}

@Controller
public class UploadTestController {
    private UploadTestService uploadTestService;

    @Autowired
    public void setUploadTestService(UploadTestService uploadTestService) {
        this.uploadTestService = uploadTestService;
    }

    @PostMapping("/test/upload/requestParam")
    public void upload(@RequestParam("file") MultipartFile multipartFile) {
        uploadTestService.upload(multipartFile);
    }

    @PostMapping("/test/uploads/requestParam")
    public void uploadList(@RequestParam("uploadFileList") List<MultipartFile> multipartFileList) {
        multipartFileList.forEach(uploadTestService::upload);
    }

    @PostMapping("/test/upload/commandObject")
    public void upload(@ModelAttribute MultipartModel multipartModel) {
        uploadTestService.upload(multipartModel.getUploadFile());
    }

    @PostMapping("/test/uploads/commandObject")
    public void uploadList(@ModelAttribute MultipartModel multipartModel) {
        multipartModel.getUploadFileList().forEach(uploadTestService::upload);
    }
}
{% endhighlight %}

## Java 파일 처리
전달 받은 `MultipartFile` 을 업로드한다.
파일명은 `getOriginalFilename` 메소드를 통해 얻을 수 있다.
`getName`이라는 메소드가 있지만 해당 메소드는 _form_ 에서 전달되는 _input tag_ 의 _name_ 이다

{% highlight java %}
@Service
public class UploadTestService {
    public void upload(MultipartFile multipartFile) {
        Path uploadTarget = Paths.get("\\upload\\" + multipartFile.getOriginalFilename());

        try (InputStream multipartFileInputStream = multipartFile.getInputStream()) {
            if (Files.notExists(uploadTarget.getParent())) {
                Files.createDirectories(uploadTarget.getParent());
            }

            Files.copy(multipartFileInputStream, uploadTarget);
        } catch (IOException e) {
            // 예외 처리
        }
    }
}
{% endhighlight %}

[sile sending form data]:https://developer.mozilla.org/ko/docs/Learn/Forms/Sending_and_retrieving_form_data#%ED%8A%B9%EB%B3%84%ED%95%9C_%EA%B2%BD%EC%9A%B0_%ED%8C%8C%EC%9D%BC_%EB%B3%B4%EB%82%B4%EA%B8%B0 "sile sending form data"
[input file]:https://developer.mozilla.org/ko/docs/Web/HTML/Element/Input/file "Input file"
[Spring mvc-multipart]:https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-multipart "Spring mvc-multipart"
[Spring file upload]:https://www.baeldung.com/spring-file-upload "Spring file upload"
[Spring mvc-multipart-forms]:https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-multipart-forms "Spring mvc-multipart-forms"