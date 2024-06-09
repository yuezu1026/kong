### Kong研究记录

# 通过直接接口调用
curl --request POST \
  --url http://127.0.0.1:8001/services/nginx/plugins \
  --header 'Content-Type: multipart/form-data' \
  --form name=jwt

![用insomnia执行](https://github.com/yuezu1026/kong/blob/main/images/image-20240609220000370.png)

