---
title: Vue+elementUI集成七牛云上传
date: 2018-03-09 10:19:27
tags: 
    - FE
    - Vue
categoties: Vue
---

> 前段时间在项目中遇到了上传图片的需求，鉴于短平快的需求，选择了七牛云作为图床，开发期间也遇到了一些小坑，现在分享一些代码和思路。

先贴代码吧

**上传组件**

``` html
<el-upload
        style="margin-top: 64px"
        action="http://upload-z2.qiniu.com"
        ref="upload"
        :multiple="false"
        :on-success="handleSuccess"
        :on-preview="handlePreview"
        :on-error="handleError"
        :before-upload="beforeUpload"
        :file-list="fileList"
        :data="postData"
        :auto-upload="false"
        list-type="picture">
    <!--<i class="el-icon-upload"></i>-->
    <el-button slot="trigger" size="small" type="primary">选取文件</el-button>
    <el-button style="margin-left: 10px;" size="small" type="success" @click="submitUpload">上传到服务器</el-button>
    <div class="el-upload__tip" slot="tip">只能上传jpg/png文件，且不超过500kb</div>

</el-upload>
```
**加密token算法**

```javascript
import CryptoJS from "crypto-js";
function utf16to8(str) {
    let out, i, len, c;
    out = "";
    len = str.length;
    for (i = 0; i < len; i++) {
        c = str.charCodeAt(i);
        if ((c >= 0x0001) && (c <= 0x007F)) {
            out += str.charAt(i);
        } else if (c > 0x07FF) {
            out += String.fromCharCode(0xE0 | ((c >> 12) & 0x0F));
            out += String.fromCharCode(0x80 | ((c >> 6) & 0x3F));
            out += String.fromCharCode(0x80 | ((c >> 0) & 0x3F));
        } else {
            out += String.fromCharCode(0xC0 | ((c >> 6) & 0x1F));
            out += String.fromCharCode(0x80 | ((c >> 0) & 0x3F));
        }
    }
    return out;
}

function utf8to16(str) {
    let out, i, len, c;
    let char2, char3;
    out = "";
    len = str.length;
    i = 0;
    while (i < len) {
        c = str.charCodeAt(i++);
        switch (c >> 4) {
            case 0:
            case 1:
            case 2:
            case 3:
            case 4:
            case 5:
            case 6:
            case 7:
                // 0xxxxxxx
                out += str.charAt(i - 1);
                break;
            case 12:
            case 13:
                // 110x xxxx 10xx xxxx
                char2 = str.charCodeAt(i++);
                out += String.fromCharCode(((c & 0x1F) << 6) | (char2 & 0x3F));
                break;
            case 14:
                // 1110 xxxx 10xx xxxx 10xx xxxx
                char2 = str.charCodeAt(i++);
                char3 = str.charCodeAt(i++);
                out += String.fromCharCode(((c & 0x0F) << 12) | ((char2 & 0x3F) << 6) | ((char3 & 0x3F) << 0));
                break;
        }
    }
    return out;
}

/*
 * Interfaces:
 * b64 = base64encode(data);
 * data = base64decode(b64);
 */
let base64EncodeChars = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-_";
let base64DecodeChars = new Array(-1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, 62, -1, -1, -1, 63,
    52, 53, 54, 55, 56, 57, 58, 59, 60, 61, -1, -1, -1, -1, -1, -1, -1, 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14,
    15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, -1, -1, -1, -1, -1, -1, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40,
    41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, -1, -1, -1, -1, -1);

function base64encode(str) {
    let out, i, len;
    let c1, c2, c3;
    len = str.length;
    i = 0;
    out = "";
    while (i < len) {
        c1 = str.charCodeAt(i++) & 0xff;
        if (i == len) {
            out += base64EncodeChars.charAt(c1 >> 2);
            out += base64EncodeChars.charAt((c1 & 0x3) << 4);
            out += "==";
            break;
        }
        c2 = str.charCodeAt(i++);
        if (i == len) {
            out += base64EncodeChars.charAt(c1 >> 2);
            out += base64EncodeChars.charAt(((c1 & 0x3) << 4) | ((c2 & 0xF0) >> 4));
            out += base64EncodeChars.charAt((c2 & 0xF) << 2);
            out += "=";
            break;
        }
        c3 = str.charCodeAt(i++);
        out += base64EncodeChars.charAt(c1 >> 2);
        out += base64EncodeChars.charAt(((c1 & 0x3) << 4) | ((c2 & 0xF0) >> 4));
        out += base64EncodeChars.charAt(((c2 & 0xF) << 2) | ((c3 & 0xC0) >> 6));
        out += base64EncodeChars.charAt(c3 & 0x3F);
    }
    return out;
}

function base64decode(str) {
    let c1, c2, c3, c4;
    let i, len, out;
    len = str.length;
    i = 0;
    out = "";
    while (i < len) {
        /* c1 */
        do {
            c1 = base64DecodeChars[str.charCodeAt(i++) & 0xff];
        } while (i < len && c1 == -1);
        if (c1 == -1) break;
        /* c2 */
        do {
            c2 = base64DecodeChars[str.charCodeAt(i++) & 0xff];
        } while (i < len && c2 == -1);
        if (c2 == -1) break;
        out += String.fromCharCode((c1 << 2) | ((c2 & 0x30) >> 4));
        /* c3 */
        do {
            c3 = str.charCodeAt(i++) & 0xff;
            if (c3 == 61) return out;
            c3 = base64DecodeChars[c3];
        } while (i < len && c3 == -1);
        if (c3 == -1) break;
        out += String.fromCharCode(((c2 & 0XF) << 4) | ((c3 & 0x3C) >> 2));
        /* c4 */
        do {
            c4 = str.charCodeAt(i++) & 0xff;
            if (c4 == 61) return out;
            c4 = base64DecodeChars[c4];
        } while (i < len && c4 == -1);
        if (c4 == -1) break;
        out += String.fromCharCode(((c3 & 0x03) << 6) | c4);
    }
    return out;
}
let safe64 = function(base64) {
    base64 = base64.replace(/\+/g, "-");
    base64 = base64.replace(/\//g, "_");
    return base64;
};
/*
* 时间戳格式化
* @param timestamp timestamp
* */
export const dateFormater = timestamp => {
    let date = new Date();
    date.setTime(timestamp);
    let y = date.getFullYear();
    let M = date.getMonth()+1 >= 10 ? date.getMonth()+1 : '0' + (date.getMonth()+1);
    let d = date.getDate() >= 10 ? date.getDate() : '0' + date.getDate();
    let h = date.getHours() >= 10 ? date.getHours() : '0' + date.getHours();
    let m = date.getMinutes() >= 10 ? date.getMinutes() : '0' + date.getMinutes();
    let s = date.getSeconds()>= 10 ? date.getSeconds() : '0' + date.getSeconds();
    return y + '-' + M + '-' + d + ' ' + h + ':' + m + ':' + s;
};

/*
* 七牛云Token生成
* */
export const getToken = (accessKey, secretKey, Policy) =>{
    let put_policy = JSON.stringify(Policy);
    let encoded = base64encode(utf16to8(put_policy));
    let hash = CryptoJS.HmacSHA1(encoded, secretKey);
    let encoded_signed = hash.toString(CryptoJS.enc.Base64);
    let upload_token = accessKey + ":" + safe64(encoded_signed) + ":" + encoded;
    return upload_token;
};
```
**上传部分**

```javascript
beforeUpload(file) {    //在图片提交前进行验证
    const _this = this;
    let accessKey = 'accessKey;
    let secretKey = 'secretKey';
    let deadline = Math.round(new Date().getTime() / 1000) + 3600;
    let policy = {"scope":"picture-private","deadline":deadline};
    _this.postData.token = getToken(accessKey,secretKey,policy)
}
```
这里我直接把accesskey和secretKe写在了代码里，实际上这是很不安全的，签名部分还是放后端完成比较好。