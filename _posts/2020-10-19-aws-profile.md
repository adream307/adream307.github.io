---
title:  "AWS 命令配置文件"
author: adream307
date:   2020-10-19 10:00:00 +0800
categories: [aws, sdk]
tags: [aws, go]
---

`aws cli` 可以配置多个 `Access Key`，在访问时手动选择使用哪个`Key`

`~/.aws/credentials`内容如下:
```ini
[default]
aws_access_key_id=AKIAIOSFODNN7EXAMPLE
aws_secret_access_key=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

[cnnw1]
aws_access_key_id=AKIAI44QH8DHBEXAMPLE
aws_secret_access_key=je7MtGbClwBF/2Zp9Utk/h3yCo8nvbEXAMPLEKEY
```

`~/.aws/config` 内容如下：
```ini
[default]
region = us-east-2
output = json 

[profile cnnw1]
region = cn-northwest-1
output = json 
```
**注意**: `config` 文件中的配置项必须添加 `profile` 前缀，而 `credentials` 文件中的配置项目则不能添加


使用`cnnw1` 查询，并且只打印 `InstanceId` 和 `State.Name`
```bash
$ aws ec2 describe-instances --profile cnnw1 --query 'Reservations[*].Instances[*].{id:InstanceId,st:State.Name}' --output text
i-0f5c2c0e3eb458c1c     stopped
i-081d62db743814df7     stopped
i-029207d69cd39531c     stopped
i-0baadca44017143c3     stopped
i-0b4671b6c02a3a728     stopped
i-09b89b21dbd9fa527     stopped
i-00879b91cf475a597     stopped
i-029053be08666bafd     stopped
i-0c9e6f2adaff0a1db     stopped
```

---

## Go SDK
`profile_exp.go`
```go
package main

import (
	"fmt"
	"github.com/aws/aws-sdk-go/aws/session"
	"github.com/aws/aws-sdk-go/service/ec2"
	"log"
)

func main() {
	sess := session.Must(session.NewSessionWithOptions(session.Options{
		Profile:           "cnnw1",
		SharedConfigState: session.SharedConfigEnable,
	}))
	cli := ec2.New(sess)

	insts, err := cli.DescribeInstances(nil)
	if err != nil {
		log.Fatal(err)
	}
	for _, r := range insts.Reservations {
		for _, i := range r.Instances {
			fmt.Printf("id :%s,state %s\n", *i.InstanceId, *i.State.Name)
		}
	}
}
```
程序运行结果如下
```txt
id :i-0f5c2c0e3eb458c1c,state stopped
id :i-081d62db743814df7,state stopped
id :i-029207d69cd39531c,state stopped
id :i-0baadca44017143c3,state stopped
id :i-0b4671b6c02a3a728,state stopped
id :i-09b89b21dbd9fa527,state stopped
id :i-00879b91cf475a597,state stopped
id :i-029053be08666bafd,state stopped
id :i-0c9e6f2adaff0a1db,state stopped

```

---
## 参考文献
- [命名配置文件](https://docs.aws.amazon.com/zh_cn/cli/latest/userguide/cli-configure-profiles.html)