# Compression Efficiency in Kafka

Kafka에 데이터를 저장할 시에 압축에 따른 효율성을 확인하기 위한 시도

## Test Environment

### 1. System
```text
CPU: Intel(R) Xeon(R) CPU E3-1231 v3 @ 3.40GHz
OS : VMware base - CentOS7
Kafka : v2.7.0

 - Producer Spec : 4 Core, 8GB Memory  
 - Kafka Spec : 4 Core, 8GB Memory
```

Kafka Compression 설정은 Topic과 Producer 모두 설정
```shell
bin/kafka-topics.sh --zookeeper kafka:2181 --replication-factor 1 --partitions 1 --create --topic pb-marshaling-and-zstd --config compression.type=zstd
```
```go
func newAsyncProducer(conf *Config, compressionType string) (sarama.AsyncProducer, error) {
	... 중략
    switch compressionType {
        case "gzip":
        saramaCfg.Producer.Compression = sarama.CompressionGZIP
        case "snappy":
        saramaCfg.Producer.Compression = sarama.CompressionSnappy
        case "lz4":
        saramaCfg.Producer.Compression = sarama.CompressionLZ4
        case "zstd":
        // zstd compression requires Version >= V2_1_0_0
        saramaCfg.Version = sarama.V2_7_0_0
        saramaCfg.Producer.Compression = sarama.CompressionZSTD
        case "":
    }
}

// Producer 생성
p, err := kafka.NewProducer("zstd")
if err != nil {
    panic(err)
}
```

Kafka `linger.ms` 는 `3초`로 설정
> sarama client에서 linger.ms 설정이 아래 설정과 같은지는 확신할 수 없으나, 비슷한 동작을 하는 것으로 보인다 
```go
saramaCfg := sarama.NewConfig()
saramaCfg.Producer.Flush.Frequency = time.Second * 3
```

### 2. Sample Data
```text
CSV 파일 사용
Rows : 50,000,000
CSV Size :  6.5GB

개별 레코드는 다른 값을 가진다
```
Output Data Json Format
```json
{
    "before_mymoney": 0,
    "send_status": 1,
    "product_cnt": 1,
    "send_cnt": 1,
    "deny_cnt": 0,
    "receive_cnt": 1,
    "send_tm":"2021-10-01T22:01:01",
    "product_mymoney": 1,
    "buy_mymoney": 1,
    "discount_rate":0,
    "buy_ipaddr": "192.168.0.2",
    "buy_login_id": 12345678,
    "buy_login_name": "홍길동",
    "receive_tm": nan
}
```
### 3. Try Compression Type
Kafka에서 효율성을 확인하기 위해 여러 압축 방식을 시도하여 데이터를 저장한다

```text
 - Json(Uncompress)
 - Json + Gzip
 - Json + Lz4
 - Json + Snappy
 - Json + Zstd
```
```text
 - ProtocolBuffer
 - ProtocolBuffer + Gzip
 - ProtocolBuffer + Lz4
 - ProtocolBuffer + Snappy
 - ProtocolBuffer + Zstd
```

### 4. Test code
- [json-marshaling-only_test.go](bench/json-marshaling-only_test.go)
- [json-marshaling-gzip_test.go](bench/json-marshaling-gzip_test.go)
- [json-marshaling-lz4_test.go](bench/json-marshaling-lz4_test.go)
- [json-marshaling-snappy_test.go](bench/json-marshaling-snappy_test.go)
- [json-marshaling-zstd_test.go](bench/json-marshaling-zstd_test.go)
- [pb-marshaling-only_test.go](bench/pb-marshaling-only_test.go)
- [pb-marshaling-gzip_test.go](bench/pb-marshaling-gzip_test.go)
- [pb-marshaling-lz4_test.go](bench/pb-marshaling-lz4_test.go)
- [pb-marshaling-snappy_test.go](bench/pb-marshaling-snappy_test.go)
- [pb-marshaling-zstd_test.go](bench/pb-marshaling-zstd_test.go)

## Result

같은 파일을 가지고 여러 압축 방식을 통해 Kafka에 저장하고, 이에 따른 저장된 사이즈, 소요시간 등과 같은 리소스 사용량을 확인

> ![topic-list](https://user-images.githubusercontent.com/31076511/142731753-d75b8dc0-1320-477c-9a68-4ab2369e857a.png)

### Size, Duration, CPU/Traffic Usage

| Compression Codec | Size(GB) | Duration(sec)  | Producer CPU(Avg) | Kafka CPU(Avg)  | Kafka Traffic(Avg/Mib)  |
| -------------- |:-----|:---------|:-----|:----|:------|
| Json(Uncomp)   | 22.7 | 432.863  | 25.8 | 9.9 | 411.8 |
| Json + Gzip    | 3.3  | 605.381  | 32.8 | 4.5 | 43    |
| Json + Lz4     | 4.6  | 326.519  | 34.6 | 8.8 | 111.8 |
| Json + Snappy  | 5.5  | 345.558  | 29   | 5.9 | 124.6 |
| Json + Zstd    | 3    | 280.466  | 37.5 | 8.1 | 86.3  |
| ProtoBuffer    | 8.5  | 272.288  | 29.1 | 5.6 | 246.7 |
| PB + Gzip      | 2.8  | 456.264  | 35.4 | 3.2 | 48.8  |
| PB + Lz4       | 3.8  | 244.481  | 34.6 | 6.6 | 122.9 |
| PB + Snappy    | 3.9  | 268.386  | 30.3 | 4.9 | 116.8 |
| PB + Zstd      | 2.6  | 220.791  | 37.2 | 8.2 | 94.8  |

### Diff Ratio
```text
Json 데이터는 uncompress json를 기준으로 비교
ProtoBuffer 데이터는 ProtoBuffer를 기준으로 비교
```

| Compression Codec | Size Ratio |Duration Ratio | Producer CPU Ratio | Kafka CPU Ratio | Traffic Ratio | Ratio SUM |
| -------------- |:-----:|:-----:|:----:|:----:|:----:|:----:|
| Json(Uncomp)   | 1.00 | 1.00 | 1.00 | 1.00 | 1.00 | 5.0 |
| Json + Gzip    | 0.15 | 1.40 | 1.27 | 0.45 | 0.10 | 3.37 |
| Json + Lz4     | 0.20 | 0.75 | 1.34 | 0.89 | 0.27 | 3.46 |
| Json + Snappy  | 0.24 | 0.80 | 1.12 | 0.60 | 0.30 | 3.06 |
| Json + Zstd    | 0.13 | 0.65 | 1.45 | 0.82 | 0.21 | 3.25 |
| ProtoBuf       | 1.00 | 1.00 | 1.00 | 1.00 | 1.00 | 5.0 |
| PB + Gzip      | 0.33 | 1.68 | 1.22 | 0.57 | 0.20 | 3.99 |
| PB + Lz4       | 0.45 | 0.90 | 1.19 | 1.18 | 0.50 | 4.21 |
| PB + Snappy    | 0.46 | 0.99 | 1.04 | 0.88 | 0.47 | 3.83 |
| PB + Zstd      | 0.31 | 0.81 | 1.28 | 1.46 | 0.38 | 4.24 |

> 다음과 같이 Ratio값을 합산하여 결과를 보는 것이 맞는지는 의문이지만, 어느정도 결과를 도출하기 위해서 사용했다

Ratio 값을 합산하면 Json/ProtoBuffer 두 기준에서 모두 Snappy 압축이 제일 좋은 성능을 가지는 것으로 보인다

실제 적용 시에는 Duration, CPU, Traffic등과 같이 중요시 되는 기준에 따라 Compression Type에 따라 선택한다

#### Encoding Compression
```shell
Zstd 압축을 사용하면 높은 Producer CPU를 사용하여 가장 낮은 디스크 사용량과 가장 낮은 처리시간을 얻을 수 있다
Gzip 압축은 Producer에서 높은 부하을 갖지만, Kafka서버에 가장 낮은 부하(CPU/Traffic/Size)를 준다
```

### Compression ratio by count
데이터 저장 수에 따른 압축율의 변화가 있을까 생각했지만, 데이터 카운트에 따른 압축율의 변화는 없는 것 같다

#### Size

| Compression Codec | 10,000 | 1,000,000 | 5,000,000 | 10,000,000 | 50,000,000 | 100,000,000 |
| -------------- |:-----:|:-----:|:----:|:----:|:----:|:----:|
| (단위) | MB | MB |  MB | GB | GB | GB |
| Json(Uncomp)   | 4.50 | 453.80 | 2355.20 | 4.50 | 22.70 | 45. 40|
| Json + Gzip    | 0.65 | 66.00 | 325.10 | 0.64 | 3.30 | 6.50 |
| Json + Lz4     | 0.92 | 93.40 | 461.80 | 0.90 | 4.60 | 9.30 |
| Json + Snappy  | 1.10 | 109.80 | 545.90 | 1.10 | 5.50 | 10.90 |
| Json + Zstd    | 0.60 | 61.40 | 301.90 | 0.59 | 3.00 | 6.10 |
| ProtoBuf       | 1.70 | 169.70 | 848.10 | 1.70 | 8.50 | 17.00 |
| PB + Gzip      | 0.56 | 57.10 | 279.90 | 0.55 | 2.80 | 5.60 |
| PB + Lz4       | 0.75 | 76.90 | 376.40 | 0.74 | 3.80 | 7.60 |
| PB + Snappy    | 0.79 | 80.00 | 394.10 | 0.77 | 3.90 | 7.90 |
| PB + Zstd      | 0.52 | 53.10 | 260.40 | 0.51 | 2.60 | 5.20 |

#### Size ratio

| Compression Codec | 10,000 | 1,000,000 | 5,000,000 | 10,000,000 | 50,000,000 | 100,000,000 |
| -------------- |:-----:|:-----:|:----:|:----:|:----:|:----:|
| Json(Uncomp)   | 1.00 | 1.00 | 1.00 | 1.00 | 1.00 | 1.00 |
| Json + Gzip    | 0.14 | 0.15 | 0.14 | 0.14 | 0.15 | 0.14 |
| Json + Lz4     | 0.20 | 0.21 | 0.20 | 0.20 | 0.20 | 0.20 |
| Json + Snappy  | 0.24 | 0.24 | 0.23 | 0.24 | 0.24 | 0.24 |
| Json + Zstd    | 0.13 | 0.14 | 0.13 | 0.13 | 0.13 | 0.13 |
| ProtoBuf       | 1.00 | 1.00 | 1.00 | 1.00 | 1.00 | 1.00 |
| PB + Gzip      | 0.33 | 0.34 | 0.33 | 0.32 | 0.33 | 0.33 |
| PB + Lz4       | 0.44 | 0.45 | 0.44 | 0.44 | 0.45 | 0.45 |
| PB + Snappy    | 0.46 | 0.47 | 0.46 | 0.45 | 0.46 | 0.46 |
| PB + Zstd      | 0.31 | 0.31 | 0.31 | 0.30 | 0.31 | 0.31 |


## Options: Resource Usage Graph 
Producer에서 Kafka로 데이터를 저장할때의 compression type별 리소스 사용량이다

### 1. Json Uncompressed
![json-uncompressed](https://user-images.githubusercontent.com/31076511/142730639-f71ac7fc-f7f7-411b-836f-4eef0d337b7e.png)


### 2. Json + Gzip
![json-gzip](https://user-images.githubusercontent.com/31076511/142730649-a56380dd-c616-474c-9c23-f6f6d6ae608c.png)


### 3. Json + Lz4
![json-lz4](https://user-images.githubusercontent.com/31076511/142730664-b68cc57b-733c-4a54-b418-f538a26e98ce.png)


### 4. Json + Snappy
![json-snappy](https://user-images.githubusercontent.com/31076511/142730677-746a5862-3c94-4b44-a6b0-1225278d12d6.png)

   
### 5. Json + Zstd
![json-zstd](https://user-images.githubusercontent.com/31076511/142730682-57714107-6323-4ea8-9324-ee16ad2e86d1.png)


### 6. Protocol Buffer
![protobuf](https://user-images.githubusercontent.com/31076511/142730705-33e1bb4b-e4c9-406e-a583-3487ab9aabc8.png)


### 7. ProtoBuf + Gzip
![pb-gzip](https://user-images.githubusercontent.com/31076511/142730708-64304d71-f593-4a06-ab83-d0d4b2be8d84.png)


### 8. ProtoBuf+ Lz4
![pb-lz4](https://user-images.githubusercontent.com/31076511/142730728-2fffb7e6-9d54-4ca6-84a9-563c174f1075.png)


### 9. ProtoBuf + Snappy
![pb-snappy](https://user-images.githubusercontent.com/31076511/142730731-2c04faa0-5234-4f48-be19-31b959e9fdd3.png)


### 10. ProtoBuf + Zstd
![pb-zstd](https://user-images.githubusercontent.com/31076511/142730740-79101cde-bf95-42c6-b804-ca13256e9f13.png)

## Reference
 - [benefits-compression-kafka-messaging](https://developer.ibm.com/articles/benefits-compression-kafka-messaging/)
 - [KIP-390:A Support Compression Level](https://cwiki.apache.org/confluence/display/KAFKA/KIP-390%3A+Support+Compression+Level)