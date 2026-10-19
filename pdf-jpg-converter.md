# AWS Hands-on Lab: Serverless PDF-to-JPG Converter

Tài liệu này hướng dẫn các bước xây dựng hệ thống Serverless cho phép người dùng upload file PDF, sau đó tự động chuyển đổi thành ảnh JPG.

## Kiến trúc Tổng quan

![Architecture](pdf-jpg-converter.png)

---

## Các bước triển khai

### Bước 1: Tạo 3 S3 Buckets
Tạo 3 bucket sau (khuyên dùng chung Region `ap-southeast-1`):
1. **Input Bucket** (`pdf-input-<suffix>`): Chặn Public Access.
2. **Output Bucket** (`jpg-output-<your-suffix>`): Chặn Public Access.
3. **Website Bucket** (`lab-web-<suffix>`): Bỏ chặn Public Access, bật **Static website hosting** (Index document: `index.html`).

### Bước 2: Tạo SQS Queue
- Tạo Standard Queue tên `pdf-convert-queue`.
- Cấu hình **Access policy** để S3 Input Bucket có thể gửi message vào queue:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3ToSendMessage",
      "Effect": "Allow",
      "Principal": {
        "Service": "s3.amazonaws.com"
      },
      "Action": "sqs:SendMessage",
      "Resource": "arn:aws:sqs:ap-southeast-1:<your-account-id>:pdf-convert-queue",
      "Condition": {
        "ArnLike": {
          "aws:SourceArn": "arn:aws:s3:::pdf-input-<your-suffix>"
        }
      }
    }
  ]
}
```

### Bước 3: Tạo IAM Roles cho Lambda

**1. `Lambda-Presign-Role` (Cho hàm Presign URL)**
Tạo IAM Role cho Lambda (Basic Execution) và thêm Inline Policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowPresignPutObject",
      "Effect": "Allow",
      "Action": ["s3:PutObject"],
      "Resource": "arn:aws:s3:::pdf-input-<your-suffix>/*"
    },
    {
      "Sid": "AllowReadOutputBucket",
      "Effect": "Allow",
      "Action": ["s3:ListBucket", "s3:GetObject"],
      "Resource": [
        "arn:aws:s3:::jpg-output-<your-suffix>",
        "arn:aws:s3:::jpg-output-<your-suffix>/*"
      ]
    }
  ]
}
```

**2. `Lambda-PDF-Convert-Role` (Cho hàm Converter)**
Tạo IAM Role cho Lambda (`AWSLambdaVPCAccessExecutionRole`) và thêm Inline Policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadFromInputBucket",
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::pdf-input-<your-suffix>/*"
    },
    {
      "Sid": "WriteToOutputBucket",
      "Effect": "Allow",
      "Action": ["s3:PutObject"],
      "Resource": "arn:aws:s3:::jpg-output-<your-suffix>/*"
    },
    {
      "Sid": "ReceiveFromSQS",
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes"
      ],
      "Resource": "arn:aws:sqs:ap-southeast-1:<your-account-id>:pdf-convert-queue"
    }
  ]
}
```

### Bước 4: Kích hoạt S3 Event Notification
- Vào Input Bucket > Properties > Event notifications.
- Tạo event `s3:ObjectCreated:Put` và `CompleteMultipartUpload` (Suffix `.pdf`), trỏ Destination về SQS Queue `pdf-convert-queue`.

### Bước 5: Setup Mạng (VPC & Endpoints)
- Tạo 1 **VPC** (vd: `10.1.0.0/16`) và 1 **Private Subnet** (không có Internet Gateway).
- Tạo **VPC Endpoint S3** (Gateway) và **VPC Endpoint SQS** (Interface) kết nối vào Private Subnet. (Lưu ý gán Security Group cho SQS Endpoint mở port 443 từ dải IP của VPC).

### Bước 6: Build Lambda Layer (CloudShell)
Chạy script sau trên AWS CloudShell để tạo Layer chứa `pdf2image` và `poppler`:

```bash
mkdir -p pdf_layer/python
cd pdf_layer

sudo dnf install -y poppler-utils

mkdir -p bin
cp $(which pdftoppm) bin/
cp $(which pdfinfo) bin/
ldd $(which pdftoppm) | awk '/=>/{print $3}' | xargs -I{} cp {} bin/ 2>/dev/null || true

pip3 install pdf2image Pillow -t python/
zip -r pdf2image_layer.zip python/ bin/

aws lambda publish-layer-version \
  --layer-name pdf2image-poppler \
  --zip-file fileb://pdf2image_layer.zip \
  --compatible-runtimes python3.12 \
  --region ap-southeast-1
```

### Bước 7: Tạo Lambda `pdf-presign-url`
- Tạo hàm Lambda, Runtime: Python 3.12, Role: `Lambda-Presign-Role`.
- **Biến môi trường**: `INPUT_BUCKET`, `OUTPUT_BUCKET`, `URL_EXPIRY` (vd: `300`).
- **Code Python**:

```python
import boto3
import os
import json

s3_client = boto3.client('s3', region_name='ap-southeast-1')
INPUT_BUCKET = os.environ['INPUT_BUCKET']
OUTPUT_BUCKET = os.environ['OUTPUT_BUCKET']
URL_EXPIRY = int(os.environ.get('URL_EXPIRY', '300'))

def lambda_handler(event, context):
    path = event.get('rawPath', '')
    params = event.get('queryStringParameters') or {}
    filename = params.get('filename', '')

    if not filename:
        return {'statusCode': 400, 'headers': cors_headers(), 'body': 'Missing filename'}

    if path == '/presign':
        presigned_url = s3_client.generate_presigned_url(
            ClientMethod='put_object',
            Params={'Bucket': INPUT_BUCKET, 'Key': filename, 'ContentType': 'application/pdf'},
            ExpiresIn=URL_EXPIRY, HttpMethod='PUT'
        )
        return {'statusCode': 200, 'headers': cors_headers(), 'body': json.dumps({'uploadUrl': presigned_url, 'filename': filename})}
        
    elif path == '/images':
        base_name = os.path.splitext(filename)[0]
        response = s3_client.list_objects_v2(Bucket=OUTPUT_BUCKET, Prefix=base_name)
        images = []
        if 'Contents' in response:
            for obj in response['Contents']:
                if obj['Key'].endswith('.jpg'):
                    img_url = s3_client.generate_presigned_url(
                        ClientMethod='get_object',
                        Params={'Bucket': OUTPUT_BUCKET, 'Key': obj['Key']}, ExpiresIn=URL_EXPIRY
                    )
                    images.append(img_url)
        return {'statusCode': 200, 'headers': cors_headers(), 'body': json.dumps({'images': images})}

    return {'statusCode': 404, 'headers': cors_headers(), 'body': 'Not Found'}

def cors_headers():
    return {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Methods': 'GET, OPTIONS',
        'Access-Control-Allow-Headers': 'Content-Type'
    }
```

### Bước 8: Tạo Lambda `pdf-to-jpg-converter`
- Tạo hàm Lambda, Runtime: Python 3.12, Role: `Lambda-PDF-Convert-Role`.
- **Gắn vào VPC** (Private Subnet).
- Tăng **Memory**: `1024 MB`, **Timeout**: `5 min`.
- **Thêm Layer** (chọn Layer ARN tạo ở Bước 6).
- **Biến môi trường**: `OUTPUT_BUCKET`, `POPPLER_PATH` (`/opt/bin`).
- Cấu hình **SQS Trigger** (chỉ định queue `pdf-convert-queue`).
- **Code Python**:

```python
import boto3
import json
import os
import tempfile
import urllib.parse
from pdf2image import convert_from_path

s3_client = boto3.client('s3')
OUTPUT_BUCKET = os.environ['OUTPUT_BUCKET']
POPPLER_PATH = os.environ.get('POPPLER_PATH', None)

def lambda_handler(event, context):
    processed = 0
    for record in event['Records']:
        body = json.loads(record['body'])
        if 'Event' in body and body['Event'] == 's3:TestEvent':
            continue

        for s3_record in body.get('Records', []):
            input_bucket = s3_record['s3']['bucket']['name']
            input_key = urllib.parse.unquote_plus(s3_record['s3']['object']['key'])
            base_name = os.path.splitext(os.path.basename(input_key))[0]
            prefix = os.path.dirname(input_key)

            with tempfile.NamedTemporaryFile(suffix='.pdf', delete=False) as tmp_pdf:
                s3_client.download_fileobj(input_bucket, input_key, tmp_pdf)
                tmp_pdf_path = tmp_pdf.name

            pages = convert_from_path(tmp_pdf_path, dpi=150, fmt='jpeg', poppler_path=POPPLER_PATH)

            for page_num, page_image in enumerate(pages, start=1):
                with tempfile.NamedTemporaryFile(suffix='.jpg', delete=False) as tmp_jpg:
                    page_image.save(tmp_jpg.name, 'JPEG', quality=90)
                    jpg_key = f"{prefix}/{base_name}_page{page_num:03d}.jpg".lstrip('/')
                    with open(tmp_jpg.name, 'rb') as f:
                        s3_client.put_object(Bucket=OUTPUT_BUCKET, Key=jpg_key, Body=f, ContentType='image/jpeg')
                    os.unlink(tmp_jpg.name)

            os.unlink(tmp_pdf_path)
            processed += 1
    return {'statusCode': 200, 'processed': processed}
```

### Bước 9: Tạo API Gateway (HTTP API)
- Tạo HTTP API, tích hợp (Integration) với Lambda `pdf-presign-url`.
- Tạo 2 Routes: `GET /presign` và `GET /images`. Gắn integration Lambda vào cả 2 route.
- Bật **CORS**: `Allow Origins: *`, `Allow Methods: GET, OPTIONS`, `Allow Headers: content-type`.
- Deploy và lấy **Invoke URL**.

### Bước 10: Cấu hình CORS trên S3 Input Bucket
- Vào S3 Input Bucket > Permissions > CORS > dán cấu hình sau:

```json
[
  {
    "AllowedHeaders": ["*"],
    "AllowedMethods": ["PUT"],
    "AllowedOrigins": ["*"],
    "ExposeHeaders": ["ETag"]
  }
]
```

### Bước 11: Cấu hình Bucket Policy trên S3 Website Bucket
- Vào S3 Website Bucket > Permissions > Bucket policy > dán:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadForWebsite",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::lab-web-<your-suffix>/*"
    }
  ]
}
```

### Bước 12: Deploy Giao diện Web
- Cập nhật biến `API_URL` trong file `index.html` của bạn bằng Invoke URL của API Gateway.
- Upload file `index.html` lên Website Bucket.
- Truy cập vào **Bucket website endpoint** để trải nghiệm.
