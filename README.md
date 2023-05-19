### 由于微信视频号没有开放平台,无法通过正常的途径获取到个人微信视频号数据,所有通过爬取数据的方式

### 微信视频号后台管理地址

#### https://channels.weixin.qq.com/login.html


### 爬取流程
1.解析登陆二维码图片的真实地址 得到例如(https://channels.weixin.qq.com/mobile/confirm_login.html?token=AQAAAOagpYoBwf-LMqwVlg)

2. 通过F12,发现二维码真实地址的token参数是由 (https://channels.weixin.qq.com/cgi-bin/mmfinderassistant-bin/auth/auth_login_code7) 接口返回

3. 如何判断此二维码是否扫描状态,接口: https://channels.weixin.qq.com/cgi-bin/mmfinderassistant-bin/auth/auth_login_status?token=AQAAAP0W70L5h5ZcdGGnqw&timestamp=1684481485761&_log_finder_uin=&_log_finder_id=&scene=7&reqScene=7

4. 如何获取扫码用户的视频: https://channels.weixin.qq.com/cgi-bin/mmfinderassistant-bin/post/post_list (必须携带)Cookie，Cookie信息需要在
auth_login_status接口的响应头中获取，这个接口需要定时调用，只有在有人扫描了这个二维码,并且确认登陆的情况下，response header 中才有cookie

### 需要在你的数据库中添加表来记录token的生成记录
```sql 
CREATE TABLE `sv_wechat_token_manage` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键',
  `token` varchar(100) NOT NULL COMMENT '微信视频号扫码token',
  `qrCode` varchar(200) NOT NULL COMMENT '二维码地址',
  `expireTime` datetime NOT NULL COMMENT '过期时间',
  `createdTime` datetime NOT NULL COMMENT '创建时间',
  `companyUuid` varchar(100) NOT NULL COMMENT '公司uuid',
  `employeeUuid` varchar(100) NOT NULL COMMENT '员工uuid',
  PRIMARY KEY (`id`) 
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='微信视频号扫码token';
```

### 调用微信接口 发送请求类

```java
package com.qf.shortVideo.service.enums;

import com.qf.shortVideo.service.api.rest.WeChatRestTemplate;
import com.qf.shortVideo.stub.bean.*;
import lombok.AllArgsConstructor;
import lombok.Getter;
import org.springframework.http.HttpMethod;
import org.springframework.http.ResponseEntity;

/**
 * @author shihao.liu
 * 微信视频号
 */
@Getter
@AllArgsConstructor
public enum WeChatAPI {
    /**
     * 获取扫码登陆TOKEN
     */
    AUTH_LOGIN_CODE("/cgi-bin/mmfinderassistant-bin/auth/auth_login_code", HttpMethod.POST,
            GetWeChatAuthTokenDTO.class),

    /**
     * 检查微信二维码是否扫码
     */
    AUTH_LOGIN_STATUS("/cgi-bin/mmfinderassistant-bin/auth/auth_login_status?token={token}", HttpMethod.POST, GetWeChatAuthLoginStatusDTO.class),


    /**
     * 获取扫码登陆用户信息
     */
    AUTH_DATA("/cgi-bin/mmfinderassistant-bin/auth/auth_data", HttpMethod.POST, WeChatUserInfoDTO.class),

    /**
     * 获取视频列表
     */
    POST_LIST("/cgi-bin/mmfinderassistant-bin/post/post_list", HttpMethod.POST, WeChatDynamicInfoDTO.class),

    /**
     * 获取账号视频数据
     */
    POST_TOTAL_DATA("/cgi-bin/mmfinderassistant-bin/statistic/new_post_total_data", HttpMethod.POST, WeChatTotalCountInfoDTO.class),

    /**
     * 获取粉丝数据
     */
    FANS_TREND("/cgi-bin/mmfinderassistant-bin/statistic/fans_trend", HttpMethod.POST, WeChatFansInfoDTO.class),

    /**
     * 根据时间获取视频列表
     */
    POST_LIST_BY_TIME("/cgi-bin/mmfinderassistant-bin/statistic/post_list", HttpMethod.POST, WeChatDynamicInfoDTO.class),

    /**
     * 某个视频近30天的数据
     */
    FEED_AGGREAGATE_DATA_BY_TAB_TYPE("/cgi-bin/mmfinderassistant-bin/statistic/feed_aggreagate_data_by_tab_type",HttpMethod.POST,WeChatFeedDataInfoDTO.class),;
    /**
     * URL
     */
    private final String uri;

    /**
     * 请求方式
     */
    private final HttpMethod httpMethod;


    /**
     * 返回类型
     */
    private final Class<?> responseType;

    /**
     * 发送不需要cookie的请求
     *
     * @param requestBody 请求参数
     * @param objects     可变参数
     */
    public <RQ, RS> ResponseEntity<RS> send(RQ requestBody, Object... objects) {
        return WeChatRestTemplate.send(this, (Class<RS>) responseType, requestBody, objects);
    }

    /**
     * 发送需要 cookie做为头
     */
    public <RQ, RS> ResponseEntity<RS> doSend(RQ requestBody) {
        return WeChatRestTemplate.doSend(this, (Class<RS>) responseType, requestBody);
    }
}
```

### WeChatRestTemplate.java
```java
package com.qf.shortVideo.service.api.rest;

import com.alibaba.fastjson.JSON;
import com.qf.shortVideo.base.configuration.WeChatConfiguration;
import com.qf.shortVideo.cache.constants.RedisKeyConstants;
import com.qf.shortVideo.cache.redis.RedisCacheWrapper;
import com.qf.shortVideo.dao.domain.AuthenticationInfo;
import com.qf.shortVideo.service.enums.WeChatAPI;
import com.qf.shortVideo.stub.enums.AuthStatusEnum;
import com.qf.shortVideo.stub.response.constants.ShortVideoResponseCode;
import com.qiaofang.common.context.LoginUserUtil;
import com.qiaofang.common.exception.BusinessException;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpHeaders;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Component;

/**
 * @author shihao.liu
 * 获取视频号数据
 */
@Slf4j
@Component
public class WeChatRestTemplate {

    private static RestTemplateWrapper restTemplateWrapper;

    private static WeChatConfiguration weChatConfiguration;

    private static RedisCacheWrapper redisCacheWrapper;


    public static <RQ, RS> ResponseEntity<RS> send(WeChatAPI api, Class<RS> responseType, RQ requestBody, Object... uriVariables) {
        HttpHeaders headers = new HttpHeaders();
        return restTemplateWrapper.execute(weChatConfiguration.getHost() + api.getUri(), api.getHttpMethod(),
                headers, responseType, JSON.toJSONString(requestBody), uriVariables);

    }

    public static <RQ, RS> ResponseEntity<RS> doSend(WeChatAPI api, Class<RS> responseType, RQ requestBody) {
        long startTime = System.currentTimeMillis();
        HttpHeaders headers = new HttpHeaders();
        // 携带头信息
        String redisKey = String.format(RedisKeyConstants.SPH_AUTH_INFO_REDIS, LoginUserUtil.getCompanyUuid(), LoginUserUtil.getUserUuid());
        log.info("获取redisKey:{},请求地址:{},", redisKey, api.getUri());
        if (!redisCacheWrapper.exists(redisKey)) {
            throw new BusinessException(ShortVideoResponseCode.NO_AUTH_SPH.getResponseCode(),
                    ShortVideoResponseCode.NO_AUTH_SPH.getResponseMessage());
        }
        AuthenticationInfo authenticationInfo = redisCacheWrapper.get(redisKey);
        if (authenticationInfo == null || AuthStatusEnum.Expired == authenticationInfo.getStatus()) {
            throw new BusinessException(ShortVideoResponseCode.NO_AUTH_SPH_EXPIRED.getResponseCode(),
                    ShortVideoResponseCode.NO_AUTH_SPH_EXPIRED.getResponseMessage());
        }
        log.info("获取redisKey:{},请求地址:{},redisValue:{}", redisKey, api.getUri(), JSON.toJSONString(authenticationInfo));
        headers.add("Cookie", authenticationInfo.getAccessToken());
        headers.add("Content-Type", "application/json");
        ResponseEntity<RS> responseEntity = restTemplateWrapper.execute(weChatConfiguration.getHost() + api.getUri(), api.getHttpMethod(), headers, responseType, JSON.toJSONString(requestBody));
        log.info("WeChatRestTemplate request,url:{},param:{},result:{},耗时:{} ms",
                weChatConfiguration.getHost() + api.getUri(), JSON.toJSONString(requestBody),
                responseEntity.getBody() == null ? null : JSON.toJSONString(responseEntity.getBody()),
                (System.currentTimeMillis() - startTime));
        return responseEntity;
    }

    @Autowired
    public void setRestTemplateWrapper(RestTemplateWrapper restTemplateWrapper) {
        WeChatRestTemplate.restTemplateWrapper = restTemplateWrapper;
    }

    @Autowired
    public void setWeChatConfiguration(WeChatConfiguration weChatConfiguration) {
        WeChatRestTemplate.weChatConfiguration = weChatConfiguration;
    }

    @Autowired
    public void setRedisCacheWrapper(RedisCacheWrapper redisCacheWrapper) {
        WeChatRestTemplate.redisCacheWrapper = redisCacheWrapper;
    }
}
```

### WeChatShortVideoService.java
```java

 public String generateAuthQrCode() {
        WeChatTokenManage tokenManage = new WeChatTokenManage();
        tokenManage.setCreatedTime(new Date());
        // 获取微信登陆token
        ResponseEntity<GetWeChatAuthTokenDTO> response = WeChatAPI.AUTH_LOGIN_CODE.send(Maps.newHashMap());
        if (Objects.isNull(response.getBody().getData()) || Objects.isNull(response.getBody().getData().getToken())) {
            throw new BusinessException(GET_WECHAT_TOKEN_ERROR.getResponseCode(), GET_WECHAT_TOKEN_ERROR.getResponseMessage());
        }
        String token = response.getBody().getData().getToken();
        tokenManage.setToken(token);
        // 生成登陆链接
        byte[] bytes = QrCodeUrlFetcher.generateBytes(token, 250);
        // 上传wos
        WosDTO wosDTO = FileOperationHelper.upload(bytes, token + ".jpg");
        tokenManage.setQrCode(wosDTO.getAccessUrl());
        Boolean flag = weChatTokenManageService.saveWeChatTokenManage(tokenManage);
        log.info("generateAuthQrCode param:{},result:{}", JSON.toJSONString(tokenManage), flag);
        return wosDTO.getAccessUrl();
    }

    public void auth() {
        List<WeChatTokenManage> data = Optional.ofNullable(weChatTokenManageService.getAllData()).orElseGet(Lists::newArrayList);
        if (CollectionUtils.isNotEmpty(data)) {
            // key = true  已扫码授权的, key = false 未扫码授权的
            Map<Boolean, List<GetWeChatAuthLoginStatusDTO>> tokenAuthMap =
                    data.stream().map(QrCodeUrlFetcher::checkLoginStatus)
                            .collect(Collectors.partitioningBy(f -> StringUtils.isNotBlank(f.getData().getCookie())));
            log.info("需要处理的token数据:{}", JSON.toJSONString(tokenAuthMap));
            // 对已经扫码登陆的用户 sv_authorization_info 更新表中数据
            List<GetWeChatAuthLoginStatusDTO> authList = tokenAuthMap.get(Boolean.TRUE);
            saveAuthorization(authList);
            // 刷新缓存
            refreshWeChatCookie(authList);
            // 更新用户头像信息
            updateWechatAuthUserInfo(authList);
            // 删除 sv_wechat_token_manage表中已经扫码授权的数据 并且expireTime<当前时间的数据
            List<Integer> ids = Stream.of(
                    Optional.ofNullable(tokenAuthMap.get(Boolean.TRUE)).orElseGet(Lists::newArrayList)
                            .stream().filter(f -> Objects.nonNull(f.getData()) && Objects.nonNull(f.getData().getId()))
                            .map(f -> f.getData().getId()),
                    Optional.ofNullable(tokenAuthMap.get(Boolean.FALSE)).orElseGet(Lists::newArrayList)
                            .stream().filter(f -> Objects.nonNull(f.getData()) && Objects.nonNull(f.getData().getId())
                                    && System.currentTimeMillis() > f.getData().getExpireTime().getTime())
                            .map(f -> f.getData().getId())
            ).flatMap(f -> f).filter(Objects::nonNull).distinct().collect(Collectors.toList());
            log.info("sv_wechat_token_manage需要删除的主键ids:{}", ids);
            weChatTokenManageService.deleteByIds(ids);
        }

    }
```
### QrCodeUrlFetcher.java
```java

package com.qf.shortVideo.service.util;

import com.baomidou.mybatisplus.core.toolkit.CollectionUtils;
import com.baomidou.mybatisplus.core.toolkit.StringPool;
import com.google.common.collect.Lists;
import com.google.common.collect.Maps;
import com.google.zxing.BarcodeFormat;
import com.google.zxing.EncodeHintType;
import com.google.zxing.common.BitMatrix;
import com.google.zxing.qrcode.QRCodeWriter;
import com.qf.shortVideo.dao.domain.WeChatTokenManage;
import com.qf.shortVideo.service.enums.WeChatAPI;
import com.qf.shortVideo.stub.bean.GetWeChatAuthLoginStatusDTO;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;

import javax.imageio.ImageIO;
import java.awt.image.BufferedImage;
import java.io.ByteArrayOutputStream;
import java.util.Collection;
import java.util.HashMap;
import java.util.List;
import java.util.stream.Collectors;

/**
 * 生成微信视频号登陆二维码
 *
 * @author shihao.liu
 */
@Slf4j
public class QrCodeUrlFetcher {

    /**
     * 最终要此链接生成二维码
     */
    private static final String AUTH_LOGIN_URL = "https://channels.weixin.qq.com/mobile/confirm_login.html?token=%s";


    /**
     * 根据token生成登陆二维码
     *
     * @param token token
     * @param size  大小
     * @return base64
     */
    public static byte[] generateBytes(String token, Integer size) {
        String url = String.format(AUTH_LOGIN_URL, token);
        return generateQrCode(url, size);
    }


    public static byte[] generateQrCode(String url, int size) {
        try {
            // 设置二维码参数
            HashMap<EncodeHintType, Object> hints = Maps.newHashMap();
            hints.put(EncodeHintType.CHARACTER_SET, "UTF-8");

            // 生成二维码矩阵
            QRCodeWriter qrCodeWriter = new QRCodeWriter();
            BitMatrix bitMatrix = qrCodeWriter.encode(url, BarcodeFormat.QR_CODE, size, size, hints);

            // 将矩阵转换为缓冲图像
            BufferedImage bufferedImage = new BufferedImage(size, size, BufferedImage.TYPE_INT_RGB);
            for (int x = 0; x < size; x++) {
                for (int y = 0; y < size; y++) {
                    int color = bitMatrix.get(x, y) ? 0xFF000000 : 0xFFFFFFFF;
                    bufferedImage.setRGB(x, y, color);
                }
            }
            // 将图像转换为字节数组
            ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
            ImageIO.write(bufferedImage, "png", outputStream);
            // 将字节数组转换为Base64编码的字符串
            return outputStream.toByteArray();
        } catch (Exception e) {
            log.error(e.getMessage(), e);
        }
        return null;
    }


    public static GetWeChatAuthLoginStatusDTO checkLoginStatus(WeChatTokenManage param) {
        ResponseEntity<GetWeChatAuthLoginStatusDTO> response = WeChatAPI.AUTH_LOGIN_STATUS.send(Maps.newHashMap(), param.getToken());
        GetWeChatAuthLoginStatusDTO auth = response.getBody();
        auth.getData().setCompanyUuid(param.getCompanyUuid());
        auth.getData().setEmployeeUuid(param.getEmployeeUuid());
        auth.getData().setId(param.getId());
        auth.getData().setToken(param.getToken());
        auth.getData().setExpireTime(param.getExpireTime());
        if (auth.getData().getStatus() == 1 &&
                auth.getData().getAcctStatus() == 1 && auth.getData().getFlag() == 1) {
            // 已扫码的情况下获取cookie
            List<String> list = response.getHeaders().get("set-cookie").stream().map(item -> Lists.newArrayList(item.split(StringPool.SEMICOLON)))
                    .flatMap(Collection::stream).filter(f -> f.contains("sessionid=") || f.contains("wxuin="))
                    .collect(Collectors.toList());
            if (!CollectionUtils.isEmpty(list)) {
                auth.getData().setCookie(String.join(StringPool.SEMICOLON, list));
            }
        }
        return auth;
    }
}
```
