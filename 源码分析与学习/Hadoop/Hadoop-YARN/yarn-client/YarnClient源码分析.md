# YarnClient源码分析

## 1. 简介

YarnClient是一个抽象类，继承AbstractService(Hadoop Service的基本实现类)。

## 2. 主要方法

**1. createYarnClient**

```java
/**
* Create a new instance of YarnClient.
*
* 为YARN Client创建一个新的实例
*/
@Public
public static YarnClient createYarnClient() {
    YarnClient client = new YarnClientImpl();
    return client;
}
```

**2. createApplication**

```java
  /**
   * <p>
   * Obtain a {@link YarnClientApplication} for a new application,
   * which in turn contains the {@link ApplicationSubmissionContext} and
   * {@link org.apache.hadoop.yarn.api.protocolrecords.GetNewApplicationResponse}
   * objects.
   * </p>
   *
   * @return {@link YarnClientApplication} built for a new application
   * @throws YarnException
   * @throws IOException
   */
  public abstract YarnClientApplication createApplication()
      throws YarnException, IOException;
```

**3. submitApplication**

```java
/**
   * <p>
   * Submit a new application to <code>YARN.</code> It is a blocking call - it
   * will not return {@link ApplicationId} until the submitted application is
   * submitted successfully and accepted by the ResourceManager.
   * </p>
   *
   * 向YARN提交一个新的application，这是个阻塞线程，直到提交的应用程序成功提交并被ResourceManager接受为止
   *
   * <p>
   * Users should provide an {@link ApplicationId} as part of the parameter
   * {@link ApplicationSubmissionContext} when submitting a new application,
   * otherwise it will throw the {@link ApplicationIdNotProvidedException}.
   * </p>
   *
   * 在提交新应用时，用户应该提供一个ApplicationId作为参数的一部分
   *
   * <p>This internally calls {@link ApplicationClientProtocol#submitApplication
   * (SubmitApplicationRequest)}, and after that, it internally invokes
   * {@link ApplicationClientProtocol#getApplicationReport
   * (GetApplicationReportRequest)} and waits till it can make sure that the
   * application gets properly submitted. If RM fails over or RM restart
   * happens before ResourceManager saves the application's state,
   * {@link ApplicationClientProtocol
   * #getApplicationReport(GetApplicationReportRequest)} will throw
   * the {@link ApplicationNotFoundException}. This API automatically resubmits
   * the application with the same {@link ApplicationSubmissionContext} when it
   * catches the {@link ApplicationNotFoundException}</p>
   *
   * 内部调用ApplicationClientProtocol.submitApplication(SubmitApplicationRequest)方法，之后，它在内部调用ApplicationClientProtocol.getApplicationReport
   * 方法等到可以确保正确提交应用程序。如果在RM保存application 状态之前，RM故障或RM重新启动，ApplicationClientProtocol.getApplicationReport(GetApplicationReportRequest)
   * 将会抛出ApplicationNotFoundException异常。
   * 此API使用相同的ApplicationSubmissionContext自动重新提交应用程序
   *
   *
   * @param appContext
   *          {@link ApplicationSubmissionContext} containing all the details
   *          needed to submit a new application
   * @return {@link ApplicationId} of the accepted application
   * @throws YarnException
   * @throws IOException
   * @see #createApplication()
   */
  public abstract ApplicationId submitApplication(
      ApplicationSubmissionContext appContext) throws YarnException,
      IOException;
```

