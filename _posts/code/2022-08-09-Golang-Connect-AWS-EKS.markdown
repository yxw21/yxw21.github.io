---
layout: post
title: "Use aws-sdk-go-v2 to link aws eks"
tags: [golang,aws,eks,aws-sdk-go-v2]
---

```
func postDataToService(namespace, service, port, path string, data []byte) ([]byte, error) {
	const (
	    region = "xxxxx"
	    key = "xxxxx"
	    secret = "xxxxx"
	    session = "xxxxx"
	    clusterName = "xxxxx"
	)
	cfg, err := config.LoadDefaultConfig(
		context.TODO(),
		config.WithRegion(region),
		config.WithCredentialsProvider(credentials.NewStaticCredentialsProvider(key, secret, session)),
	)
	if err != nil {
		return nil, err
	}
	awsClient := eks.NewFromConfig(cfg)
	describeClusterOutput, err := awsClient.DescribeCluster(context.TODO(), &eks.DescribeClusterInput{
		Name: aws.String(clusterName),
	})
	if err != nil {
		return nil, err
	}
	preSignClient := sts.NewPresignClient(sts.NewFromConfig(cfg))
	preSignedURLRequest, err := preSignClient.PresignGetCallerIdentity(context.TODO(), &sts.GetCallerIdentityInput{}, func(preSignOptions *sts.PresignOptions) {
		preSignOptions.ClientOptions = append(preSignOptions.ClientOptions, func(stsOptions *sts.Options) {
			stsOptions.APIOptions = append(stsOptions.APIOptions, smithyhttp.SetHeaderValue("x-k8s-aws-id", clusterName))
			stsOptions.APIOptions = append(stsOptions.APIOptions, smithyhttp.SetHeaderValue("X-Amz-Expires", "60"))
		})
	})
	if err != nil {
		return nil, err
	}
	bearerToken := "k8s-aws-v1." + base64.RawURLEncoding.EncodeToString([]byte(preSignedURLRequest.URL))
	ca, err := base64.StdEncoding.DecodeString(*describeClusterOutput.Cluster.CertificateAuthority.Data)
	if err != nil {
		return nil, err
	}
	clientSet, err := kubernetes.NewForConfig(
		&rest.Config{
			Host:        *describeClusterOutput.Cluster.Endpoint,
			BearerToken: bearerToken,
			TLSClientConfig: rest.TLSClientConfig{
				CAData: ca,
			},
		},
	)
	if err != nil {
		return nil, err
	}
	restClient := clientSet.RESTClient()
	return restClient.Post().Body(data).RequestURI("/api/v1/namespaces/"+ namespace +"/services/" + service + ":"+ port +"/proxy" + path).DoRaw(context.TODO())
}
```
