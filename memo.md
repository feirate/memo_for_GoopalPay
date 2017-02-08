# memo
## 2017/02/07
+ pay-trade中使用的模板

**1. 抽象泛型类：** APITemplate<T> 接口基础模板

``` JAVA
/**
 * Created by hongzong.li on 8/28/16.
 */
public abstract class APITemplate<T> { // 抽象泛型类
    protected Logger logger = LogManager.getLogger(getClass());

 protected abstract T process() throws BizException; // 抽象接口的业务处理过程,业务错误封装成业务类异常并抛出

 public APIResponse<T> execute()  { // 通过调用execute方法执行
        try {
            T result = process(); // 执行业务处理过程并取得结果,捕获过程中产生的异常
 logger.info("Response: " + result);
 return APIResponse.buildSuccess(result); // 处理成功调用接口响应类编辑响应结果并返回
 } catch (BizException e) { // 处理业务类异常
            logger.error("Biz exception", e);
 logger.info("Response: " + e.getMessage());
 if (e.getErrorCode() != null) {
                return APIResponse.buildFail(e.getErrorCode());
 } else {
                return APIResponse.buildFail();
 }
        } catch (IllegalArgumentException e) { // 处理非法参数异常
            logger.error("Illegal argument exception", e);
 logger.info("Response: " + e.getMessage());
 return APIResponse.buildFail(ErrorCodes.PARAM_ERROR.withErrorMsg(e.getMessage()));
 } catch (Throwable e) { // 处理其他异常
            logger.error("Unexpected exception", e);
 logger.info("Response: " + e.getMessage());
 return APIResponse.buildFail();
 }
    }
}
```

示例: 应用信息查询接口
``` JAVA
/**
 * 应用信息查询接口：根据查询条件返回相应应用信息集合
 * 入参：MercPtlAppClctReqParams
 * 出参：APIResponse<List<MercPtlAppClctRspParams>>
 */
@Override
public APIResponse<List<MercPtlAppClctRspParams>> getMercPtlAppInfo(
        MercPtlAppClctReqParams mPtlAppReq) {
    return new APITemplate<List<MercPtlAppClctRspParams>>() { // 创建一个APITemplate类型的实例,重写业务处理过程process()方法
        @Override
        protected List<MercPtlAppClctRspParams> process() throws BizException {
            return mercPtlQuService.getMercPtlAppInfo(mPtlAppReq);
        }
    }.execute(); // 调用execute()方法执行重写的业务处理过程
}
```

**2. 泛型类：** APIResponse<T> 接口响应模板

``` JAVA
/**
 * 接口响应模板：所有API接口通用响应模板
 */
public class APIResponse<T> implements Serializable {
    private static final long serialVersionUID = 2499006212770361270L; // 序列化版本ID,
    private static final String SUCCESS_CODE = "000000";
    private static final String SUCCESS_MSG = "成功";
    private static final String INTERNAL_ERROR_CODE = "999999";
    private static final String INTERNAL_ERROR_MSG = "系统繁忙, 请稍后再试";
    protected String version; // 响应模板系统数据(版本号,返回码,返回内容,签名串)与业务数据(泛型变量)
    protected String returnCode;
    protected String returnMsg;
    protected String sign;
    protected T data;
    private static final String DEFAULT_VERSION = "1.0";
    private static final APIResponse<Object> SUCCESS_RESPONSE = new APIResponse("000000", "成功"); // 私有静态常量
    private static final APIResponse<Object> ERROR_RESPONSE = new APIResponse("999999", "系统繁忙, 请稍后再试"); //

    public APIResponse() { // 根据需要创建多个响应模板,多态构造器
        this.version = "1.0";
        this.returnCode = "000000";
        this.returnMsg = "成功";
    }

    public APIResponse(String version, String returnCode, String returnMsg, T data, String sign) {
        this.version = "1.0";
        this.returnCode = "000000";
        this.returnMsg = "成功";
        this.version = version;
        this.returnCode = returnCode;
        this.returnMsg = returnMsg;
        this.sign = sign;
        this.data = data;
    }

    public APIResponse(String returnCode, String returnMsg, T data, String sign) {
        this.version = "1.0";
        this.returnCode = "000000";
        this.returnMsg = "成功";
        this.sign = sign;
        this.returnCode = returnCode;
        this.returnMsg = returnMsg;
        this.data = data;
    }

    public APIResponse(String returnCode, String returnMsg, T data) {
        this.version = "1.0";
        this.returnCode = "000000";
        this.returnMsg = "成功";
        this.returnCode = returnCode;
        this.returnMsg = returnMsg;
        this.data = data;
    }

    public APIResponse(String returnCode, String returnMsg) {
        this.version = "1.0";
        this.returnCode = "000000";
        this.returnMsg = "成功";
        this.returnCode = returnCode;
        this.returnMsg = returnMsg;
    }

    public APIResponse(ErrorCode error) {
        this(error.getErrorCode(), error.getErrorMsg(), (Object)null); // this指向同一个类中不同参数列表的另外一个构造器,必须是第一条语句
    }

    public static <T> APIResponse<T> buildFail(String returnCode, String returnMsg) { // 多态静态方法创建响应模板实例
        return new APIResponse(returnCode, returnMsg);
    }

    public static <T> APIResponse<T> buildFail(ErrorCode error) {
        return buildFail(error.getErrorCode(), error.getErrorMsg());
    }

    public static <T> APIResponse<T> buildFail() {
        return ERROR_RESPONSE;
    }

    public static <T> APIResponse<T> buildSuccess(T data) {
        return new APIResponse("000000", "成功", data);
    }

    public static <T> APIResponse<T> buildSuccess(T data, String sign) {
        return new APIResponse("000000", "成功", data, sign);
    }

    public static <T> APIResponse<T> buildSuccess() {
        return SUCCESS_RESPONSE;
    }

    public boolean successful() { // 提供常用工具方法
        return StringUtils.equals(this.returnCode, SUCCESS_RESPONSE.getReturnCode());
    }
```

**3. 泛型类：** BizTemplate<T> 业务处理模板

``` JAVA
/**
 * Created by hongzong.li on 7/12/16.
 * 业务处理模板：所有的业务处理逻辑使用的共同模板。包括校验,逻辑处理,结果处理，处理反馈(回调等)
 */
public abstract class BizTemplate<T> {
    protected Logger logger = LogManager.getLogger(getClass());

    private final String monitorKey; // 定义成员变量-监控项目的引用

    private static final String ERROR_POSTFIX = "_error";

    protected abstract void checkParams(); // 抽象方法定义校验过程,具体业务实例必须实现

    protected abstract T process() throws BizException; // 抽象方法定义逻辑处理过程,具体业务实例必须实现.过程中出现的问题作为业务异常抛出

    protected void postProcess() {  // 普通方法定义处理反馈,不是必须实现
    }

    protected void onSuccess() { // 普通方法定义成功结果处理,不是必须实现
    }

    protected void onFailure() { // 普通方法定义成功失败处理,不是必须实现
    }

    public BizTemplate() {
        this(null); // 需要在每个构造器内给final成员变量初始化
    }

    public BizTemplate(String monitorKey) {
        this.monitorKey = monitorKey; // 需要在每个构造器内给final成员变量初始化
    }

    public T execute() throws BizException { // 执行业务处理逻辑
        long timeStart = System.currentTimeMillis(); // 开始记录业务处理时长
        boolean exceptional = true;
        try { // 定义业务处理逻辑,捕获过程中出现的异常并分情况处理
            checkParams();
            T result = process();
            onSuccess();
            exceptional = false;
            return result;
        } catch (IllegalArgumentException e) {
            logger.warn("Illegal argument exception", e);
            onFailure();
            throw new BizException(ErrorCodes.PARAM_ERROR.withErrorMsg(e.getMessage()));
        } catch (ValidateException e) {
            logger.warn("Validate exception", e);
            onFailure();
            if (e.getErrorCode() != null) {
                throw new BizException(e.getErrorCode());
            }
            throw new BizException(e);
        } catch (UncheckedServiceException e) {
            logger.error("Service exception", e);
            onFailure();
            if (e.getErrorCode() != null) {
                throw new BizException(e.getErrorCode(), e);
            }
            throw new BizException(e);
        } catch (BizException e) {
            onFailure();
            logger.error("Biz exception", e);
            throw e;
        } catch (Throwable e) {
            logger.error("Unexpected exception", e);
            onFailure();
            throw new BizException(ErrorCodes.INTERNAL_ERROR, e);
        } finally { // 在try的return执行之前进行监控记录和处理反馈的操作
            recordMonitorIfNecessary(monitorKey, exceptional, System.currentTimeMillis() - timeStart);
            postProcess();
        }
    }

    private void recordMonitorIfNecessary(String monitorKey, boolean exceptional, long duration) {
        if (StringUtils.isEmpty(monitorKey)) {
            return;
        }

        GMonitor monitor = MonitorUtils.getInstance();
        if (exceptional) {
            monitor.mark(monitorKey + ERROR_POSTFIX); // 记录异常时监控指标
        } else {
            monitor.timing(monitorKey, duration); // 记录业务正常处理结束需要时长
        }
    }

}
```
