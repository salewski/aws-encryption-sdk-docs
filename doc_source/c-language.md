# AWS Encryption SDK for C<a name="c-language"></a>


|  | 
| --- |
|   The AWS Encryption SDK for C is a preview release\. The code and the documentation are subject to change\.  | 

The AWS Encryption SDK for C is designed to provide a client\-side encryption library for developers who are writing applications in C, and to serve as a foundation for implementations of the AWS Encryption SDK in higher\-level programming languages\.

Like all implementations of the AWS Encryption SDK, the AWS Encryption SDK for C offers advanced data protection features\. These include [envelope encryption](https://docs.aws.amazon.com/crypto/latest/userguide/cryptography-concepts.html#define-envelope-encryption), additional authenticated data \(AAD\), and secure, authenticated, symmetric key [algorithm suites](concepts.md#crypto-algorithm), such as 256\-bit AES\-GCM with key derivation and signing\.

All language\-specific implementations of the AWS Encryption SDK are fully interoperable\. For example, you can encrypt data with the AWS Encryption SDK for C and decrypt it with [any supported language implementation](programming-languages.md), including the [AWS Encryption CLI](crypto-cli.md)\.

The AWS Encryption SDK for C uses the AWS SDK for C\+\+ to interact with AWS Key Management Service \(AWS KMS\) so it can support the optional [KMS keyring](choose-keyring.md#use-kms-keyring)\. However, the AWS Encryption SDK doesn't require AWS KMS or any other AWS service\.

For details about programming with the AWS Encryption SDK, see the [C examples](c-examples.md), the [examples](https://github.com/awslabs/aws-encryption-sdk-c/tree/master/examples) in the aws\-encryption\-sdk\-c repository on GitHub, and the [C API documentation](https://awslabs.github.io/aws-encryption-sdk-c/html/)\.

**Topics**
+ [Install and Build](c-language-installation.md)
+ [Using the C SDK](c-language-using.md)
+ [Choosing a Keyring](choose-keyring.md)
+ [Examples](c-examples.md)