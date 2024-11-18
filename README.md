
在网上购物时候，不止可以通过名称搜索商品，也可以拍照上传图片搜索商品。比如某宝上拍个图片就能搜索到对应的商品。


![](https://img2024.cnblogs.com/blog/2448954/202411/2448954-20241118092506220-164884692.png)


腾讯、阿里都提供了类似的图像搜索服务，这类服务原理都差不多：


* 在一个具体的图库上，新增或者删除图片。
* 通过图片搜索相似的图片。


本文对接的是**腾讯云的图像搜索**。


# 添加配置


添加 maven 依赖：



```
<dependency>
    <groupId>com.tencentcloudapigroupId>
    <artifactId>tencentcloud-sdk-javaartifactId>
    <version>3.1.1129version>
dependency>

```

引入配置：



```
tencentcloud:
  tiia:
    secretId: ${SECRET_ID}
    secretKey: ${SECRET_KEY}
    endpoint: tiia.tencentcloudapi.com
    region: ap-guangzhou
    groupId: test1

```

secretId 和 secretKey 都是在 [API秘钥](https://github.com) 地址：`https://console.cloud.tencent.com/cam/capi`，groupId 是图库 id。


配置 bean



```
@Data
@Configuration
@ConfigurationProperties("tencentcloud.tiia")
public class TencentCloudTiiaProperties {

    private String secretId;

    private String secretKey;

    private String endpoint = "tiia.tencentcloudapi.com";

    private String region = "ap-guangzhou";

    private String groupId;

}

```


```
@Configuration
@ConditionalOnClass(TencentCloudTiiaProperties.class)
public class TencentCloudTiiaConfig {

    @Bean
    public TiiaClient tiiaClient(TencentCloudTiiaProperties properties) {
        Credential cred = new Credential(properties.getSecretId(), properties.getSecretKey());
        HttpProfile httpProfile = new HttpProfile();
        httpProfile.setEndpoint(properties.getEndpoint());
        ClientProfile clientProfile = new ClientProfile();
        clientProfile.setHttpProfile(httpProfile);
        TiiaClient client = new TiiaClient(cred, properties.getRegion(), clientProfile);
        return client;
    }
}


```

tiiaClient 是搜图的核心，在后面新增、删除、搜索图片都会使用到。


# 图库更新


新建图库之后，需要将图片批量的导入到图库中。一般开始会批量将上架的图片批量导入到图片库，一般只需要操作一次。


商品有修改、新增、下架操作时，图片也需要有对应的更新操作。但是每次都更新都同步更新操作，可能会导致数据库频繁更新，服务器压力增加，需要改成，每次更新图片后，同步到缓存中，然后定时处理缓存的数据：


![](https://img2024.cnblogs.com/blog/2448954/202411/2448954-20241118092524532-1363482880.png)


腾讯图像搜索没有图像更新接口，只有图像删除和新增的接口，那就**先调用删除，再调用新增的接口**


## 删除图片


图片删除调用 `tiiaClient.DeleteImages` 方法，主要注意请求频率限制。



> 默认接口请求频率限制：10次/秒


这里就简单处理，使用线程延迟处理 `Thread.sleep(100)`，删除图片只要指定 EntityId：



```
@Data
public class DeleteImageDTO {

    private String entityId;

    private List<String> picName;
}

```

如果指定 PicName 就删除 EntityId 下面的具体的图片，如果不指定 PicName 就删除整个 EntityId。


删除图片代码如下：



```
public void deleteImage(List list) {
    if (CollectionUtils.isEmpty(list)) {
        return;
    }
    list.stream().forEach(deleteImageDTO -> {
        List picNameList = deleteImageDTO.getPicName();
        if (CollectionUtils.isEmpty(picNameList)) {
            DeleteImagesRequest request = new DeleteImagesRequest();
            request.setGroupId(tiiaProperties.getGroupId());
            request.setEntityId(deleteImageDTO.getEntityId());
            try {
                // 腾讯限制qps
                Thread.sleep(100);
                tiiaClient.DeleteImages(request);
            } catch (TencentCloudSDKException | InterruptedException e) {
                log.error("删除图片失败, entityId {} 错误信息 {}", deleteImageDTO.getEntityId(), e.getMessage());
            }
        } else {
            picNameList.stream().forEach(picName -> {
                DeleteImagesRequest request = new DeleteImagesRequest();
                request.setGroupId(tiiaProperties.getGroupId());
                request.setEntityId(deleteImageDTO.getEntityId());
                request.setPicName(picName);
                try {
                    Thread.sleep(100);
                    tiiaClient.DeleteImages(request);
                } catch (TencentCloudSDKException | InterruptedException e) {
                    log.error("删除图片失败, entityId {}, 错误信息 {}", deleteImageDTO.getEntityId(), picName, e.getMessage());
                }
            });
        }
    });

}

```

## 新增图片


新增图片调用 `tiiaClient.CreateImage` 方法，这里也需要注意调用频率的限制。除此之外还有两个限制：


* 限制图片大小不可超过 5M
* 限制图片分辨率不能超过分辨率不超过 4096\*4096


既然压缩图片需要耗时，那就每次上传图片先压缩一遍，这样就能解决调用频率限制的问题。压缩图片引入 thumbnailator：



```
<dependency>
    <groupId>net.coobirdgroupId>
    <artifactId>thumbnailatorartifactId>
    <version>0.4.20version>
dependency>

```

压缩工具类：



```
/**
 * 图片压缩
 * @param url             图片url
 * @param scale           压缩比例
 * @param targetSizeByte  压缩后大小 KB
 * @return
 */
public static byte[] compress(String url, double scale, long targetSizeByte) {
    if (StringUtils.isBlank(url)) {
        return null;
    }
    long targetSizeKB = targetSizeByte * 1024;
    try {
        URL u = new URL(url);
        double quality = 0.8;
        ByteArrayOutputStream outputStream = new ByteArrayOutputStream(1024);
        do {
            Thumbnails.of(u).scale(scale) // 压缩比例
                    .outputQuality(quality) // 图片质量
                    .toOutputStream(outputStream);
            long fileSize = outputStream.size();
            if (fileSize <= targetSizeKB) {
                return outputStream.toByteArray();
            }
            outputStream.reset();
            if (quality > 0.1) {
                quality -= 0.1;
            } else {
                scale -= 0.1;
            }
        } while (quality > 0 || scale > 0);
    } catch (IOException e) {
        log.error(e.getMessage());
    }
    return null;
}

```

通过缩小图片尺寸和降低图片质量将图片压缩到固定的大小，这里都会先压缩一遍。解决调用频率限制的问题。


限制图片的分辨率也是使用到 thumbnailator 里面的 size 方法。



> thumbnailator 压缩图片和限制大小，不能一起使用，只能分来调用。


设置尺寸方法：



```
/**
 * 图片压缩
 * @param imageData
 * @param width           宽度
 * @param height          高度
 * @return
 */
public static byte[] compressSize(byte[] imageData,String outputFormat,int width,int height) {
    if (imageData == null) {
        return null;
    }
    ByteArrayInputStream inputStream = new ByteArrayInputStream(imageData);
    try {
        BufferedImage bufferedImage = ImageIO.read(inputStream);
        int imageWidth = bufferedImage.getWidth();
        int imageHeight = bufferedImage.getHeight();
        if (imageWidth <= width && imageHeight <= height) {
            return imageData;
        }
        ByteArrayOutputStream outputStream = new ByteArrayOutputStream(1024);
        Thumbnails.of(bufferedImage)
                .size(width,height)
                .outputFormat(outputFormat)
                .toOutputStream(outputStream);
        return outputStream.toByteArray();
    } catch (IOException e) {
        log.error(e.getMessage());
    }
    return null;
}

```

这里的 width 和 height 并不是直接设置图片的长度和长度，而是不会超过这个长度和宽度。如果有一个超过限制大小，压缩尺寸，长宽比保持不变。


新增图片需要指定 EntityId、url 以及 picName。



```
@Data
public class AddImageDTO {

    private String entityId;

    private String imgUrl;

    private String picName;

}

```

解决了图片压缩问题，上传图片就比较简单了：



```
public void uploadImage(List list) {
    if (CollectionUtils.isEmpty(list)) {
        return;
    }
    list.stream().forEach(imageDTO -> {
        String imgUrlStr = imageDTO.getImgUrl();
        if (StringUtils.isBlank(imgUrlStr)) {
            // 跳过当前元素
            return;
        }
        CreateImageRequest request = new CreateImageRequest();
        request.setGroupId(tiiaProperties.getGroupId());
        request.setEntityId(imageDTO.getEntityId());
        String imageUrl = imageDTO.getImgUrl();
        // 限制大小
        byte[] bytes = ImageUtils.compress(imageUrl,0.6,1024 * 5);
        String imageFormat = imageUrl.substring(imageUrl.lastIndexOf(".") + 1);
        // 限制分辨率
        bytes = ImageUtils.compressSize(bytes,imageFormat,4096,4096);
        request.setImageBase64(new String(Base64.encodeBase64(bytes), StandardCharsets.UTF_8));
        //JSONObject tagJson = new JSONObject();
        //tagJson.put("code","搜索字段");
        //request.setTags(JSONObject.toJSONString(tagJson));
        request.setPicName(imageDTO.getPicName());
        try {
            tiiaClient.CreateImage(request);
        } catch (TencentCloudSDKException e) {
            log.error("图像上传失败 error:{}", e.getMessage());
        }
    });
}

```

Tags 图片自定义标签，设置图片的参数，搜索的时候就可以根据参数搜索到不同的图片。


## 更新图片


一般商品更新，将数据存入缓存中：



```
  String value = "demo key";
  SetOperations<String, Object> opsForSet = redisTemplate.opsForSet();
  opsForSet.add(RedisKeyConstant.PRODUCT_IMAGE_SYNC_CACHE_KEY, value);

```

再定时执行任务：



```
public void syncImage() {
    while (true) {
        SetOperations<String, Object> operations = redisTemplate.opsForSet();
        Object obj = operations.pop(RedisKeyConstant.PRODUCT_IMAGE_SYNC_CACHE_KEY);
        if (obj == null) {
            log.info("暂未发现任务数据");
            return;
        }
        String pop = obj.toString();
        if (StringUtils.isBlank(pop)) {
            continue;
        }
        DeleteImageDTO deleteImageDTO = new DeleteImageDTO();
        deleteImageDTO.setEntityId(pop);
        try {
            this.deleteImage(Collections.singletonList(deleteImageDTO));
        } catch (Exception e) {
            log.error("删除图片失败,entityId {}",pop);
        }
        // todo 获取数据具体的数据
        String imageUrl="";
        // todo picName 需要全局唯一
        String picName="";

        AddImageDTO addImageDTO = new AddImageDTO();
        addImageDTO.setEntityId(pop);
        addImageDTO.setImgUrl(imageUrl);
        addImageDTO.setPicName(picName);
        try {
            this.uploadImage(Collections.singletonList(addImageDTO));
        } catch (Exception e) {
            log.error("上传图片失败,entityId {}",pop);
        }


    }
}

```

`operations.pop` 从集合随机取出一个数据并移除数据，先删除图片，再从数据库中查询是否存在数据，如果存在就新增图片。


# 搜索图片


图像搜索调用 `tiiaClient.SearchImage` 方法，需要传图片字节流，压缩图片需要文件后缀。



```
@Data
public class SearchRequest {

  private byte[] bytes;

  private String suffix;

}

```


```


public ImageInfo [] analysis(SearchRequest searchRequest) throws IOException, TencentCloudSDKException {
    SearchImageRequest request = new SearchImageRequest();
    request.setGroupId(tiiaProperties.getGroupId());
    // 筛选，对应上传接口 Tags
    //request.setFilter("channelCode=\"" + searchRequest.getChannelCode() + "\"");、
    byte[] bytes = searchRequest.getBytes();
    bytes = ImageUtils.compressSize(bytes,searchRequest.getSuffix(),4096,4096);
    String base64 = Base64.encodeBase64String(bytes);
    request.setImageBase64(base64);
    SearchImageResponse searchImageResponse = tiiaClient.SearchImage(request);
    return searchImageResponse.getImageInfos();
}

```

根据返回的 ImageInfos 数组获取到 EntityId，就能获取对应的商品信息了。


# 总结


对接图像搜索，主要是做图像的更新和同步操作。相对于每次更新就同步接口，这种方式对于服务器的压力也比较大，**先将数据同步到缓存中，然后在定时的处理数据**，而搜索图片对于数据一致性相对比较宽松，分散库写入的压力。


新增图片使用 thumbnailator 压缩图片和缩小图片，对于调用请求频率限制，新增图片每次都会压缩一次图片，每次压缩时间大概都大于 100ms，解决了请求频率限制的问题。而删除图片，就简单使用线程休眠的方式休眠 100ms。


做好图片更新的操作之后，搜索图库使用 `tiiaClient.SearchImage` 方法就能获取到对应的结果信息了。


[Github示例](https://github.com):[蓝猫机场](https://fenfang.org)


`https://github.com/jeremylai7/springboot-learning/blob/master/springboot-test/src/main/java/com/test/controller/ImageSearchController.java`


