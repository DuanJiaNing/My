随着流量的增加，GCP App Engine 会自动为应用分配更多的资源，但自动分配资源仍然受到一些阀值的约束，其中一条便是：发送到应用的请求，请求体不能大于32M。而对于一些上传大文件的需求，这个限制使得那些将文件上传服务的 EndPoint 设置在 App Engine 上的应用无法正常处理请求。

考虑到 App Engine 不允许应用操作本地存储，而且我们上传的文件一般也不会保存在本地，而是存到 Cloud Storage，File Store 等地方。

这里以 Cloud Storage 为例，Cloud Storage 提供了[Signed URLs](https://cloud.google.com/storage/docs/access-control/signed-urls)(除了失效时间之外，可以不受任何约束)访问 Storage 中的资源。这种方式在进行签名时需要将`Content-Type`用作签名的一部分，然而对于`mutlipart/formdata`类型的表单，Content-Tyep 是 `mutlipart/formdata  ---boundaryString`的形式，而 boundaryString 是动态生成的，这就导致[无法为文件上传请求进行签名](https://stackoverflow.com/questions/38752334/google-storage-with-signed-url)。

在这个 [questions](https://stackoverflow.com/questions/38752334/google-storage-with-signed-url) 中也可以看到有人给出了解决方案，那就是使用 [post-object#policydocument](https://cloud.google.com/storage/docs/xml-api/post-object#policydocument)。

看如下的示例：
```html
<form action="https://storage.googleapis.com/bucket-name/file-name" method="post" enctype="multipart/form-data">
    <input type="hidden" name="GoogleAccessId" value="1234567890123@developer.gserviceaccount.com">
    <input type="hidden" name="policy" value="eyJleHBpcmF0aW9uIjogIjIwMTAtMDYtMTZUMTE6MTE6MTFaIiwNCiAi">
    <input type="hidden" name="signature" value="BSAMPLEaASAMPLE6SAMPLE+SAMPPLEqSAMPLEPSAMPLE+SAMPLEgSAMPL">
    <input type="hidden" name="success_action_status" value="200">
    <input type="file" name="file">
    <input type="submit" value="Upload">
</form>
```
`policy` 字段包含一个指明用户需要遵循的上传规则，即 policy document，经过 base64 编码后填入 policy 字段。

```json
// policy document
{
  "expiration": "2010-06-16T11:11:11Z",
  "conditions": [
    ["content-length-range", 0, 1073741824],
    ["eq", "$success_action_status", "200"]
  ]
}
```
上面的 `content-length-range`指明用户上传文件大小限制在 0~1GB 之间，此外还有其他几种规则可以添加，需要注意的是，这种方式只支持一次上传一个文件。form 的 `signature` 字段是签名之后的 policy document，需要使用 service account key 进行签名，签名算法为 SHA256withRSA。go 的实现如下:

```go

import (
	"crypto"
	"crypto/rand"
	"crypto/rsa"
	"crypto/sha256"
	"crypto/x509"
	"encoding/base64"
	"encoding/json"
	"encoding/pem"
)

func signPolicy(policy string) (string, error) {
	secretKey, err := loadSecretKey()
	if err != nil {
		return "", err
	}

	// sign base64ed policy document using RSA with SHA-256 using a secret key
	d := sha256SumMessage(base64.StdEncoding.EncodeToString([]byte(policy)))
	messageDigest, err := rsa.SignPKCS1v15(rand.Reader, secretKey, crypto.SHA256, d)
	if err != nil {
		return "", err
	}

	return base64.StdEncoding.EncodeToString(messageDigest), nil
}

func sha256SumMessage(msg string) []byte {
	h := sha256.New()
	h.Write([]byte(msg))
	d := h.Sum(nil)
	return d
}


func loadSecretKey() (priKey *rsa.PrivateKey, err error) {
	// RSA private key, extract from service account key json file, private_key filed
	blockPri, _ := pem.Decode([]byte(`-----BEGIN PRIVATE KEY-----
-----END PRIVATE KEY-----
`))

	// may returns a *rsa.PrivateKey, a *ecdsa.PrivateKey, or a ed25519.PrivateKey
	// see doc here: https://golang.org/src/crypto/x509/pkcs8.go
	prkI, err := x509.ParsePKCS8PrivateKey(blockPri.Bytes)
	if err != nil {
		return nil, err
	}

	return prkI.(*rsa.PrivateKey), err
}
```
在 GCP 的 [IAM & admin#Service account](https://console.cloud.google.com/iam-admin/serviceaccounts) 中可手动为 App Engine default service account 创建 RSA key，选择把私钥以 json 文件的格式下载到本地。文件格式如下:
```json
{
  "type": "service_account",
  "project_id": "***",
  "private_key_id": "***",
  "private_key": "-----BEGIN PRIVATE KEY-----***-----END PRIVATE KEY-----\n",
  "client_email": "***",
  "client_id": "***",
  "auth_uri": "***",
  "token_uri": "***",
  "auth_provider_x509_cert_url": "***",
  "client_x509_cert_url": "***"
}
```
其中 private_key 的值即为 RSA 私钥。需要注意的是，把私钥直接放在代码中是很不安全的，接下来需要做的就是找个安全的地方来保存并定期更替这些私钥。[KMS](https://cloud.google.com/kms) 是个可选的方案。