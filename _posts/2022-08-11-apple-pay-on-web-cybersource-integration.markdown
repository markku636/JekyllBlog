---
layout: post
title: Apple Pay on Web + Cybersource 串接筆記
date:  2022-08-11 01:01:01 +0800
image: apple-pay.svg
categories: Payment
tags: Apple Pay Cybersource integrate Fontend NET MVC
description : Apple Pay on Web + Cybersource 串接筆記
author : Mark ku
---

## 目的
因工作需要美國及德國專案需要界接 apple pay。

## Apple Pay 原理
[參考 啾啾鞋影片](https://www.youtube.com/watch?v=ksFXEY6P_ec) 

## 美國有多少人口使用 apple pay
[參考 statista 網站](https://www.oberlo.com/statistics/how-many-people-use-apple-pay)

## Apple pay / Google pay 和第三方支付的差異
Apple pay / Google pay 和第三方支付最大的不同是，第三方金流公司會協助處理和銀行帳務問題，但 Apple pay / Google pay 並不會。

在 Apple 官方的[成功案例中](https://developer.apple.com/apple-pay/payment-platforms/)，自己對接 apple pay的公司規模都相當的大，大多數都是透過 payment provider，我猜主要因為大部分的銀行並沒有這麼標準及各國法規都不太一樣，各家銀行如果資料交換失敗，要處理的帳務問題就會很多，處理這段的問題是一般公司無法負擔的，我們在美國的金流商，則是採用 cyber source。

## 網頁如何發起支付 
早期各家瀏覽器都是各自載入 js lib 去實作，後面 w3c 網站瀏覽器的對支付訂義標準規格，Safari 及 Chrome 都己實作，PaymentRequest api。
![](https://i.imgur.com/foyz78G.png)
(相容性)
* 實測 window.PaymentRequest，一定要 https ，否則會在瀏覽器中，找不到這物件。
* Google pay 只能在 chrome 上用，Apple pay 只能在 safari 上使用 ( desktop and mobile )
* 但 Apple Pay Payment Request API ，官方範例感覺有缺漏，目前仍需要用 apple lib 進行發起支付

## 程式串接前需提前準備的項目
* 付款頁需要 Https 環境 ( dev、prod )
* 蘋果電腦及 iPhone
* 商店需自行申辦 Apple Developer 開發者帳號 ( 99 USD / Year )
* 在開發者後台商戶域名通過驗證
* 在開發者後台上傳 金流商 CSR 及開發者 CSR 至蘋果後台
* Apple pay Button 及 相關 logo 需符合 [Apple UI 規範 ](https://developer.apple.com/apple-pay/marketing/)

## Apple Pay 付款流程
當使用者按下付款按鈕 >  前端 Call 後端 api 去和蘋果驗證金流商戶，並建立交易的 session，取得用戶端 token > 此時 iphone 會請求使用者刷臉或指紋驗證 > 透過金流商去建立訂單。

## 後台設定並取得憑證
### 建立金流商戶 ( Merchant )
#### 至[蘋果開發者後台](https://developer.apple.com/account)> Certificates, Identifiers & Profiles
![](https://i.imgur.com/g2r9RP9.png)

#### Identifiers > App IDs > Merchant IDs > Identifiers + 
![](https://i.imgur.com/MrdtgwQ.png)

#### Merchant IDs > Continue
![](https://i.imgur.com/SBvgCVx.png)

#### Name的部份隨便 key，輸入商戶識別 id  Identifier (官方建議  {domainName} + {appName })，請先記錄起來，之後程式串接時要傳遞。
![](https://i.imgur.com/DoR07Pv.png)

### 接著，從這畫面，我們得知正式撰寫金流程式前必須先準備，以下三項
![](https://i.imgur.com/3O8GjP7.png)

#### 一、上傳金流商(CyberSource 的 CSR)
##### Cybersource's BackOffice > Payment Configuration > Apple Pay > Configure > 填入 Apple Merchant ID > Generate New Certificate Signing Request > 下載憑證 > 上傳至蘋果後台 Apple Pay Payment Processing on the Web

#### 二、透過自己的蘋果電腦產生 CSR，去蘋果後台產生憑證
##### Keychain Access > Request a Certificate from Certifcate Authority...
![](https://i.imgur.com/ajCvShl.png)

##### 2.輸入 CA 相關資訊
![](https://i.imgur.com/PgytYqf.png) 
##### 3. 選擇憑證格式 ( 預設 RSA 即可 )
![](https://i.imgur.com/HazTLCl.png)

##### 4.至開發人員後台 > Apple Pay Merchant Identity Certificate  > 將剛產生的 CSR 上傳上去

##### 5.上傳完後會有個 Download，按下後會將憑證下載至電腦，之後商戶驗證時和蘋果發出請求需要憑證，但 net 元件 x509 無法使用 Cer，所以要透過 apple 電腦轉成 p12 檔。
![](https://i.imgur.com/px1rkOO.png)

##### 6. 產生 P12 檔 for 後端程式商戶驗證
將下載下來的 cer 拖進 Keychain Access 的 loign, 剛拖進來會發現憑證是 certificate is not trusted。
![](https://i.imgur.com/TAw4e5H.png)

此時到 [APPLE PKI 網站](https://www.apple.com/certificateauthority/ )    
安裝圖中紅框選取的憑證後，剛安裝憑證就會變成 This certificate vaild.  
![](https://i.imgur.com/IPe2gG3.png)

右鍵 > Export ( 密碼可隨便輸 )
![](https://i.imgur.com/2vLsC4Z.png) 

![](https://i.imgur.com/jk1RrRn.png)

##### 三、驗證域名是不是自己的
輸入驗證域名 > 下載驗證檔案 > 放在該台網站伺服器 > 按下 Verify按鈕  
![](https://i.imgur.com/ob8Lwks.png)

## 撰寫程式
可以參考蘋果[官方的 Apple Pay Live Demo](https://applepaydemo.apple.com/)，可以從範例程式得知，使用　apple pay 的前端主要的事件流程有:
* onvalidatemerchant ( 使用者按下按鈕，至自己的後端驗證商戶 )  
* onpaymentauthorized  ( 商戶驗證成功，觸發交易 )
* onpaymentmethodselected ( 付款方式選擇 )
* onshippingcontactselected ( 選擇收人時觸發 )
* onshippingmethodselected ( 選擇運輸方式)

### 前端範例程式
```
<script src="https://applepay.cdn-apple.com/jsapi/v1/apple-pay-sdk.js"></script>

<style>
    apple-pay-button {
        --apple-pay-button-width: 150px;
        --apple-pay-button-height: 30px;
        --apple-pay-button-border-radius: 3px;
        --apple-pay-button-padding: 0px 0px;
        --apple-pay-button-box-sizing: border-box;
    }
</style>

<h1>Apple Pay Welcome</h1>
<h2>Apple pay button is only show in safari!!! </h2>

<apple-pay-button buttonstyle="black" type="plain" locale="en" onclick="onApplePayButtonClicked()">123</apple-pay-button>

<script>
    function onApplePayButtonClicked() {

        if (!ApplePaySession) {
            return;
        }

        // Define ApplePayPaymentRequest
        const request = {
            "countryCode": "US",
            "currencyCode": "USD",
            "merchantCapabilities": [
                "supports3DS"
            ],
            "supportedNetworks": [
                "visa",
                "masterCard",
                "amex",
                "discover"
            ],
            "total": {
                "label": "Ibuypower",
                "type": "final",
                "amount": "0.1"
            }
        };

        // Create ApplePaySession
        const session = new ApplePaySession(3, request);
        const failObject = {
            'status': ApplePaySession.STATUS_FAILURE
        }

        session.onvalidatemerchant = event => {
            const validationURL = event.validationURL;

            const failObject = {
                'status': ApplePaySession.STATUS_FAILURE
            }

            getApplePaySession(validationURL).then(function (response) {
                debugger

                let result = JSON.parse(response)
                session.completeMerchantValidation(result);
            }).then(function (response) {
                session.completeMerchantValidation(failObject)
            }).catch(err => {
                session.completeMerchantValidation(failObject)
            })
        };
    
session.onpaymentauthorized = event => {

            /*alert('onpaymentauthorized' + JSON.stringify(event.payment.token.paymentData))*/

            var paymentDataString =
                JSON.stringify(event.payment.token.paymentData);
            var paymentDataBase64 = btoa(paymentDataString);

            debugger
            let data = {
                amount: request.total.amount,
                paymentTokenObject: paymentDataBase64
            }

            paymentProcess(data).then(function (response) {
     
                if (response === true) {
                    /*alert('true')*/
                    const result = {
                        "status": ApplePaySession.STATUS_SUCCESS
                    };
                    session.completePayment(result);
                } else {
                    session.completePayment(failObject);
                }
                //let result = JSON.parse(response)
                //session.completeMerchantValidation(result);
            }).then(function (response) {
                session.completeMerchantValidation(failObject)
            }).catch(err => {
                session.completeMerchantValidation(failObject)
            })

            // Define ApplePayPaymentAuthorizationResult

        };    

        session.oncancel = event => {
            alert('oncancel')
            session.abort(); // maybe not*/            
        };

        session.begin();
    }

    // 驗證商戶
    function getApplePaySession(url) {

        return new Promise(function (resolve, reject) {

            var xhr = new XMLHttpRequest();
            xhr.open('POST', '/applepay/ValidateMerchant');
            xhr.onload = function () {
                if (this.status >= 200 && this.status < 300) {
                    resolve(JSON.parse(xhr.response));
                } else {
                    reject({
                        status: this.status,
                        statusText: xhr.statusText
                    });
                }
            };

            xhr.onerror = function () {
                reject({
                    status: this.status,
                    statusText: xhr.statusText
                });
            };

            xhr.setRequestHeader("Content-Type", "application/json");
            xhr.send(JSON.stringify({ validationUrl: url }));
        });
    }


    function paymentProcess(data) {
        return new Promise(function (resolve, reject) {

            var xhr = new XMLHttpRequest();
            xhr.open('POST', '/applepay/paymentProcess');
            xhr.onload = function () {
                if (this.status >= 200 && this.status < 300) {
                    debugger
                    resolve(JSON.parse(xhr.response));
                } else {
                    reject({
                        status: this.status,
                        statusText: xhr.statusText
                    });
                }
            };

            xhr.onerror = function () {
                reject({
                    status: this.status,
                    statusText: xhr.statusText
                });
            };

            xhr.setRequestHeader("Content-Type", "application/json");
            xhr.send(JSON.stringify(data));
        });
    }
</script>
```
### 後端程式 ( NET MVC)
```
 /// <summary>
      /// 商戶驗證
      /// </summary>
      [HttpPost]
      public JsonResult ValidateMerchant(VerifyMerchantRequest request) {
         string strResult = string.Empty;
         try {
            
            ServicePointManager.SecurityProtocol = SecurityProtocolType.Tls12;
            ServicePointManager.Expect100Continue = false;

            System.Net.ServicePointManager.SecurityProtocol = System.Net.SecurityProtocolType.Tls12;
            /* Merchant Identity憑證 */
            string certPath = Request.MapPath(@"~/App_Data/ApplePay.p12"); //Merchant Identifier憑證路徑
            string certPwd = "123"; //Merchant Identifier憑證密碼
            X509Certificate2 cert = new X509Certificate2(certPath, certPwd, X509KeyStorageFlags.MachineKeySet);

            /* 建立PayLoad */
            var payload = new {
               displayName = "letgo",  // 名稱
               initiative = "web", // 網頁
               initiativeContext = "adm.letgo.com.tw", // 域名
               merchantIdentifier = "merchant.letgo.com.tw.testPayment", // 商戶號
            };

            string strPayLoad = JsonConvert.SerializeObject(payload);

            /* 將Payload以POST方式拋送至Apple提供的validationURL */
            /* HTTP Request需以Merchant Identity憑證送出 */
            /* 驗證成功後，Apple將會回傳Merchant Session物件*/

            #region HTTP Web Result

            HttpWebRequest httpRequest = (HttpWebRequest)HttpWebRequest.Create(request.ValidationUrl);

            httpRequest.Method = WebRequestMethods.Http.Post;
            httpRequest.ContentType = "application/json";
            httpRequest.ContentLength = strPayLoad.Length;

            httpRequest.ClientCertificates.Add(cert);

            using (StreamWriter sw = new StreamWriter(httpRequest.GetRequestStream())) {
               sw.Write(strPayLoad);
               sw.Flush();
               sw.Close();
            }

            HttpWebResponse response = httpRequest.GetResponse() as HttpWebResponse;

            using (StreamReader sr = new StreamReader(response.GetResponseStream(), Encoding.UTF8)) {
               strResult = sr.ReadToEnd();
               sr.Close();
            }

            #endregion HTTP Web Result
         }
         catch (Exception ex) {
         }
         finally {
         }

         /* 將Merchant Session物件回應至Client端*/
         return Json(strResult);
      }

      /// <summary>
      /// 付款
      /// </summary>
      /// <param name="paymentProcessRequest"></param>
      /// <returns></returns>
      [HttpPost]
      public async JsonResult PaymentProcess(PaymentProcessRequest)
      {
      // 和你的金流商串接，呼叫你的金流商 api 
      }
   }
```

## 遇到比較特別的問題
### 後端請求和蘋果在商戶驗證時，出現 The underlying connection was closed: An unexpected error occurred on a send 錯誤
錯誤的憑證蘋果的 api gateway 不會回應你，請檢查商戶或域名驗證、請求時帶的憑證，傳遞的 Payload 一定要正確。

### 和 Cybersource 建立定單時，出現 Invalid_Request，並指定paymentInformation.fluidData.value  欄位錯誤
![](https://i.imgur.com/j6wfsV6.png)
cybersource沒有提供 apple pay的測試環境，請直接用正式環境，進行開發

### 引入 apple js 時 type script error 
```
npm install @types/applepayjs --save --dev
```

## 參考資料
### [cybersource 交易狀態碼](https://support.cybersource.com/knowledgebase/Knowledgearticle/?code=000001630)
### [Apple Pay 官方網站](https://developer.apple.com/apple-pay/planning/)
### [关于Apple Pay接入和开发，看这一篇就够了](https://zhuanlan.zhihu.com/p/45068888)
### [立即富線上金流 apple pay 串接文件](https://www.paynow.com.tw/applepay/PayNow_ApplePay_v1.0.5.pdf)
### [Radial Payments & Fraud Documentation](https://docs.radial.com/ptf/Content/Topics/payments/apple-pay-web.htm)
### [院長的系統開發大小事](https://ianwu.tw/press/programming/third_party/integrate_apple_pay_on_web.html#%E5%8F%83%E8%80%83%E8%B3%87%E6%96%99)
### [參考 React 範例程式](https://github.com/google-pay/google-pay-button/tree/main/src/button-react)
### [綠界科技 APPLE PAY 金流介接 - NET 範例程式](https://github.com/ECPay/ApplePay_NET)
### [站內付 2.0 - 串接文件](https://www.ecpay.com.tw/Content/files/gw_701.pdf)