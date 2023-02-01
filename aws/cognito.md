## Dịch vụ Amazon Cognito
**Amazon Cognito** cung cấp giải pháp cho việc xác thực (authentication), uỷ quyền (authorization), quản lý user (user management) cho ứng dụng web hoặc đi động. 

Có thể login trực tiếp bằng username, password, hoặc thông qua 1 bên thứ 3 như Facebook, Amazon, Google hoặc Apple.

Hai thành phần chính của **Amazon Cognito** là **user pools** and **identity pools**.
- User pools : Thư mục user cung cấp lựa chọn sign-up, sign-in cho user.
- Identity pools: Cho phép bạn cấp quyền cho user để access các dịch vụ khác của AWS 

## Kịch bản sử dụng  user pool và identity pool

	
![scenario-cup-cib2](https://docs.aws.amazon.com/images/cognito/latest/developerguide/images/scenario-cup-cib2.png)

1. Người dùng đăng nhập thông qua **user pool** và nhận về 1 **user pool tokens** sau khi đăng nhập thành công
2. Tiếp theo ứng dụng trao đổi **user pool tokens** vừa nhận được để lấy **AWS credentials** thông qua **identity pool**
3. Cuối cùng, người dùng ứng dụng có thể dùng **AWS credentials** này cho việc access các dịch vụ AWS như Amazon S3 hoặc DynamoDB

## Các tính năng của Amazon Cognito

### User pools

- Sign-up, sign in
- Có built-in, customizable web UI cho việc sign in.
- Đăng nhập mạng xã hội bằng Facebook, Google, Đăng nhập bằng Amazon và Đăng nhập bằng Apple cũng như thông qua các nhà cung cấp danh tính SAML và OIDC từ nhóm người dùng của bạn.
- Quản lý thư mục và hồ sơ người dùng.
- Các tính năng bảo mật như xác thực đa yếu tố (MFA), kiểm tra thông tin đăng nhập bị xâm phạm, bảo vệ tiếp quản tài khoản cũng như xác minh qua điện thoại và email.
- Quy trình công việc tùy chỉnh và migration người dùng thông qua  AWS Lambda.

### Identity pools

Với **Identity pools**, người dùng có thể lấy thông tin xác thực AWS tạm thời để truy cập các dịch vụ AWS, chẳng hạn như Amazon S3 và DynamoDB

## Kết luận
Đối với dự án SNSCrawling chúng ta sẽ dùng theo kịch bản này

### Upload CSV

![Access AWS services with a user pool and an identity pool](https://docs.aws.amazon.com/images/cognito/latest/developerguide/images/scenario-cup-cib.png)

Setting role để phân quyền cho user
```yaml
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "mobileanalytics:PutEvents",
                "cognito-sync:*",
                "cognito-identity:*"
            ],
            "Resource": [
                "*"
            ]
        },
        {
            "Sid": "tensult56a5c0ff4916A",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:DeleteObject"
            ],
            "Resource": [
                "arn:aws:s3:::bucket-name/${cognito-identity.amazonaws.com:sub}/*"
            ]
        },
        {
            "Sid": "tensult56a5c0ff4916B",
            "Effect": "Allow",
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::bucket-name"
        }
    ]
}
```
Code
###Get token
```
	AWSCognito.config.region = 'ap-northeast-1'; // リージョン
	AWSCognito.config.credentials = new AWS.CognitoIdentityCredentials({
	    IdentityPoolId: $IdentityPoolId,
	});
	var authenticationData = {
        Username: $email,
        Password: $password
    };
    var authenticationDetails = new AmazonCognitoIdentity.AuthenticationDetails(authenticationData);
    
    var userData = {
        Username: $email,
        Pool: userPool
    };
    var cognitoUser = new AmazonCognitoIdentity.CognitoUser(userData);

    //authen proccess
	cognitoUser.authenticateUser(authenticationDetails, {
        onSuccess: function (result) {
            alert("Login success");
            var idToken = result.getIdToken().getJwtToken();          
            var accessToken = result.getAccessToken().getJwtToken();
            var refreshToken = result.getRefreshToken().getToken();
        },
 
        onFailure: function(err) {
            console.log(err);
        }
    }
```
###AWS service access
```js
    AWS.config.region = 'ap-northeast-1'; // リージョン
    AWS.config.credentials = new AWS.CognitoIdentityCredentials({
        IdentityPoolId: $IdentityPoolId,
        Logins: {
            'cognito-idp.ap-northeast-1.amazonaws.com/$UserPoolID': $idToken
        }
    });


    AWS.config.credentials.get(function(err) {    
        if (err) return console.error(err);
        else console.log(AWS.config.credentials);
                                                                                                                                                                                                        
        var s3 = new AWS.S3({
            apiVersion: '2006-03-01',
            params: {Bucket: 'bucket-name'}
        });

        s3.putObject({ 
            Key: AWS.config.credentials.identityId + '/' + f.name,
            ContentType: f.type,
            Body: $('#ms_word_filtered_html').val(),
        }, function(err, data) {
            if (err) {
              return alert("There was an error creating your album: " + err.message);
            }
            alert("Successfully upload file.");
        });

        s3.putObject({ 
            Key: "not-permissions" + '/' + f.name,
            ContentType: f.type,
            Body: $('#ms_word_filtered_html').val(),
        }, function(err, data) {
            if (err) {
              return alert("There was an error creating your album: " + err.message);
            }
            alert("Successfully upload file.");
        });
    });

```

### Call API

![Access resources with API Gateway and Lambda with a user pool](https://docs.aws.amazon.com/images/cognito/latest/developerguide/images/scenario-api-gateway.png)

Tham khảo:

### Signin screen

[Amazon Cognitoを使ったサインイン画面]
(https://www.tdi.co.jp/miso/amazon-cognito-sign-up)

https://ldarren.medium.com/access-aws-s3-with-cognito-7c96fd1038f0
https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_examples_s3_cognito-bucket.html