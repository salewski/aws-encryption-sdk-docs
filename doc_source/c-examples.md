# AWS Encryption SDK for C Examples<a name="c-examples"></a>


|  | 
| --- |
|   The AWS Encryption SDK for C is a preview release\. The code and the documentation are subject to change\.  | 

The following examples show you how to use the AWS Encryption SDK for C to encrypt and decrypt data\. 

When you install and build the AWS Encryption SDK for C, the source code for these and other examples are included in the `examples` subdirectory, and they are compiled and built into the `build` directory\. You can also find them in the [examples](https://github.com/awslabs/aws-encryption-sdk-c/tree/master/examples) subdirectory of the [aws\-encryption\-sdk\-c](https://github.com/awslabs/aws-encryption-sdk-c/) repository on GitHub\. 

**Topics**
+ [Encrypting and Decrypting Strings](#c-example-strings)

## Encrypting and Decrypting Strings<a name="c-example-strings"></a>

The following example shows you how to use the AWS Encryption SDK for C to encrypt and decrypt a string\. 

This example features the KMS [keyring](concepts.md#keyring), a type of keyring that uses an [AWS Key Management Service \(AWS KMS\)](https://aws.amazon.com/kms/) customer master key \(CMK\) to generate and encrypt data keys\. The example includes some code written in C\+\+ because the AWS Encryption SDK for C uses the AWS SDK for C\+\+ to call AWS KMS\. 

For help creating a CMK, see [Creating Keys](https://docs.aws.amazon.com/kms/latest/developerguide/create-keys.html) in the *AWS Key Management Service Developer Guide*\. For help finding the Amazon Resource Name \(ARN\) of an existing CMK, see [Finding the Key ID and ARN](https://docs.aws.amazon.com/kms/latest/developerguide/viewing-keys.html#find-cmk-id-arn) in the *AWS Key Management Service Developer Guide*\.

**See the complete code sample**: [string\.cpp](https://github.com/awslabs/aws-encryption-sdk-c/blob/master/examples/string.cpp)

**Topics**
+ [Encrypt a String](#c-example-string-encrypt)
+ [Decrypt a String](#c-example-string-decrypt)

### Encrypt a String<a name="c-example-string-encrypt"></a>

The first part of this example uses a KMS keyring with one CMK to encrypt a plaintext string\. 

Step 1: Construct the keyring\.  
Create a KMS keyring and configure it with the Amazon Resource Name \(ARN\) of an AWS KMS CMK\. The keyring in this example is configured with one CMK, but you can configure a KMS keyring with multiple CMKs, including CMKs in different AWS Regions and different accounts\.  
When you create a keyring with multiple CMKs, you specify the CMK that is used to generate and encrypt the plaintext data key, and an optional array of additional CMKs that encrypt the same plaintext data key\. In this case, we specify only the generator CMK\.   
Before running this code, replace the example CMK ARN with a valid one\.  

```
const char * key_arn = "arn:aws:kms:us-west-2:111122223333:key/1234abcd-12ab-34cd-56ef-1234567890ab";    

struct aws_cryptosdk_keyring *kms_keyring = 
       Aws::Cryptosdk::KmsKeyring::Builder().Build(key_arn);
```

Step 2: Create a CMM\.  
Create a [cryptographic materials manager](concepts.md#crypt-materials-manager) \(CMM\) that uses the keyring\. This example uses the default CMM provided in the SDK, but you can use any supported CMM\. It also uses the default allocator provided by the [AWS C Common library](https://github.com/awslabs/aws-c-common)\.   
After you create a CMM with the keyring, you can release your reference to the keyring using the method that the SDK provides\. The CMM retains a reference to the keyring object during its lifetime, and references to the CMM and keyring objects are released when you destroy the CMM\. This pattern helps to prevent memory leaks and to prevent the objects from being released while they are in use\.  

```
struct aws_allocator *alloc = aws_default_allocator();

struct aws_cryptosdk_cmm *cmm = aws_cryptosdk_default_cmm_new(alloc, kms_keyring);

// When you add the keyring to the CMM, release the keyring object
aws_cryptosdk_keyring_release(kms_keyring);
```

Step 3: Create a session\.  
Create a session using the allocator, a mode enumerator, and the CMM\.  
Every session requires a mode: either `AWS_CRYPTOSDK_ENCRYPT` to encrypt or `AWS_CRYPTOSDK_DECRYPT` to decrypt\. To change the mode of an existing session, use the `aws_cryptosdk_session_reset` method\.  
After you create a session with the CMM, you can release your reference to the CMM using the method that the SDK provides\. The session retains a reference to the CMM and keyring objects during its lifetime, and references to all three objects are released when you destroy the session\. This pattern helps to prevent memory leaks and to prevent the objects from being released while they are in use\.  

```
struct aws_cryptosdk_session *session = 
       aws_cryptosdk_session_new_from_cmm(alloc, AWS_CRYPTOSDK_ENCRYPT, cmm);

// When you add the CMM to the session, release the CMM object
aws_cryptosdk_cmm_release(cmm);
```

Step 4: Set the size of the plaintext data\.  
Use the `aws_cryptosdk_session_set_message_size` method to tell the SDK the exact size of the plaintext string\. This method call isn't necessary when decrypting, because the SDK gets the ciphertext length from the [encrypted message](concepts.md#message)\.  

```
const char *plaintext_input = "Hello world!";
const size_t plaintext_len_input = strlen(plaintext_input);

aws_cryptosdk_session_set_message_size(session, plaintext_len_input)
```

Step 5: Set the encryption context\.  
An [encryption context](concepts.md#encryption-context) is arbitrary, non\-secret additional authenticated data\. When you provide an encryption context on encrypt, the AWS Encryption SDK cryptographically binds the encryption context to the ciphertext so that the same encryption context is required to decrypt the data\. Using an encryption context is optional, but we recommend it as a best practice\.  
First, create a hash table that includes the encryption context strings\.  

```
// Allocate a hash table for the encryption context.
int set_up_enc_ctx(struct aws_allocator *alloc, struct aws_hash_table *my_enc_ctx) 

// Create encryption context strings
AWS_STATIC_STRING_FROM_LITERAL(enc_ctx_key1, "Example");
AWS_STATIC_STRING_FROM_LITERAL(enc_ctx_value1, "String");
AWS_STATIC_STRING_FROM_LITERAL(enc_ctx_key2, "Company");
AWS_STATIC_STRING_FROM_LITERAL(enc_ctx_value2, "MyCryptoCorp");

// Put the key-value pairs in the hash table
aws_hash_table_put(my_enc_ctx, enc_ctx_key1, (void *)enc_ctx_value1, &was_created)
aws_hash_table_put(my_enc_ctx, enc_ctx_key2, (void *)enc_ctx_value2, &was_created)
```
Get a mutable pointer to the encryption context in the session\. Then, use the `aws_cryptosdk_enc_ctx_clone` function to copy the encryption context into the session\. We keep the copy in `my_enc_ctx` so we can validate the value after decrypting the data\.  
The encryption context is part of the session, not a parameter passed to the session process function\. This guarantees that the same encryption context is used for every segment of a message, even if the session process function is called multiple times to encrypt the entire message\.  

```
struct aws_hash_table *session_enc_ctx = aws_cryptosdk_session_get_enc_ctx_ptr_mut(session);

aws_cryptosdk_enc_ctx_clone(alloc, session_enc_ctx, my_enc_ctx)
```

Step 6: Encrypt the string\.  
To encrypt the plaintext string, use the `aws_cryptosdk_session_process` method with the session in encryption mode\.  
When encrypting, the plaintext fields are input fields; the ciphertext fields are output fields\. When the processing is complete, the `ciphertext_output` field contains the [encrypted message](concepts.md#message), including the actual ciphertext, encrypted data keys, and the encryption context\. You can decrypt this encrypted message by using the AWS Encryption SDK in any supported programming language\.  

```
// Gets the length of the plaintext that the session processed
size_t plaintext_consumed_output;

aws_cryptosdk_session_process(session,
                              ciphertext_output,
                              ciphertext_buf_sz_output,
                              ciphertext_len_output,
                              plaintext_input,
                              plaintext_len_input,
                              &plaintext_consumed_output)
```

Step 7: Clean up the session\.  
The final steps verify that the session processing is complete and that all of the plaintext is processed\. Then, they destroy the session, including the references to the CMM and the keyring\.  
If you prefer, instead of destroying the session, you can reuse the session with the same keyring and CMM to decrypt the string, or to encrypt or decrypt other messages\. To use the session for decrypting, use the `aws_cryptosdk_session_reset` method to change the mode to `AWS_CRYPTOSDK_DECRYPT`\.  

```
if (!aws_cryptosdk_session_is_done(session)) {
    aws_cryptosdk_session_destroy(session);
    return 8;
}

if (plaintext_consumed != plaintext_len) abort();
aws_cryptosdk_session_destroy(session);
```

### Decrypt a String<a name="c-example-string-decrypt"></a>

The second part of this example decrypts an encrypted message that contains the ciphertext of the original string\. 

Step 1: Construct the keyring\.  
When you decrypt data in AWS KMS, you pass in the [encrypted message](concepts.md#message) that the encrypt API returned\. The [Decrypt API](https://docs.aws.amazon.com/kms/latest/APIReference/API_Decrypt.html) doesn't take a CMK as input\. Instead, AWS KMS uses the same CMK to decrypt the ciphertext that it used to encrypt it\. However, the AWS Encryption SDK lets you specify a KMS keyring with CMKs on encrypt and decrypt\.  
On decrypt, you can configure a keyring with only the CMKs that you want to use to decrypt the encrypted message\. For example, you might want to create a keyring with only the CMK that is used by a particular role in your organization\. The AWS Encryption SDK will never use a CMK unless it appears in the decryption keyring\. If the SDK can't decrypt the encrypted data keys by using the CMKs in the keyring that you provide, either because none of CMKs in the keyring were used to encrypt any of the data keys, or because the caller doesn't have permission to use the CMKs in the keyring to decrypt, the decrypt call fails\.  
In this example, we specify a keyring that is configured with the same CMK that was used to encrypt the string\. Before running this code, replace the example CMK ARN with a valid one\.  

```
const char * key_arn = "arn:aws:kms:us-west-2:111122223333:key/1234abcd-12ab-34cd-56ef-1234567890ab"    

struct aws_cryptosdk_keyring *kms_keyring =
        Aws::Cryptosdk::KmsKeyring::Builder().Build(key_arn);
```

Step 2: Create a CMM\.  
Create a [cryptographic materials manager](concepts.md#crypt-materials-manager) \(CMM\) that uses the keyring\. This example uses the default CMM provided in the SDK, but you can use any supported CMM\. It also uses the default allocator provided by the [AWS C Common library](https://github.com/awslabs/aws-c-common)\.   
After you use the keyring to create a CMM, you can release your reference to the keyring using the method that the SDK provides\. The CMM retains a reference to the keyring object during its lifetime, and both objects are released when you destroy the CMM\. This pattern helps to prevent memory leaks and to prevent the objects from being released while they are in use\.  

```
struct aws_allocator *alloc = aws_default_allocator();

struct aws_cryptosdk_cmm *cmm = aws_cryptosdk_default_cmm_new(alloc, kms_keyring);

// When you add the keyring to the CMM, release the keyring object
aws_cryptosdk_keyring_release(kms_keyring);
```

Step 3: Create a session\.  
Create a session using the allocator and the CMM\. To configure the session for decryption, configure the session with the `AWS_CRYPTOSDK_DECRYPT` mode\.   
After you create a session with a CMM, you can release your reference to the CMM using the method that the SDK provides\. The session retains a reference to the CMM and keyring objects during its lifetime, and all three objects are released when you destroy the session\. This pattern helps to prevent memory leaks and to prevent the objects from being released while they are in use\.  

```
struct aws_cryptosdk_session *session =
	aws_cryptosdk_session_new_from_cmm(alloc, AWS_CRYPTOSDK_DECRYPT, cmm);

// When you add the CMM to the session, release the CMM object
aws_cryptosdk_cmm_release(cmm);
```

Step 4: Decrypt the string\.  
To decrypt the string, use the `aws_cryptosdk_session_process` method with the session that is configured for decryption\.   
When decrypting, the ciphertext fields are input fields and the plaintext fields are output fields\. The `ciphertext_input` field holds the [encrypted message](message-format.md) that the encrypt method returned\. When the processing is complete, the `plaintext_output` field contains the plaintext \(decrypted\) string\.  

```
size_t ciphertext_consumed_output;

aws_cryptosdk_session_process(session,
                              plaintext_output,
                              plaintext_buf_sz_output,
                              plaintext_len_output,
                              ciphertext_input,
                              ciphertext_len_input,
                              &ciphertext_consumed_output)
```

Step 6: Confirm that the process is complete\.  
After decrypting, the code runs `aws_cryptosdk_session_is_done`, which confirms that the session processing is complete and that the signature has been verified\. If it returns FALSE, the code destroys the session to prevent memory leaks\. Also, if any ciphertext remains in the buffer, the example stops\.  
Be sure that the session is done, especially on decrypt\. The entire plaintext might be returned before the signature is verified\.  

```
if (!aws_cryptosdk_session_is_done(session)) {
        aws_cryptosdk_session_destroy(session);
        return 13;
}

if (ciphertext_consumed != ciphertext_len) abort();
```

Step 7: Verify the encryption context\.  
Be sure that the actual encryption context — the one that was used to decrypt the message — contains the encryption context that you provided when encrypting the message\. The actual encryption context might include extra pairs, because the [cryptographic materials manager](concepts.md#crypt-materials-manager) \(CMM\) can add pairs to the provided encryption context before encrypting the message\.  
In the AWS Encryption SDK in C, you are not required to provide an encryption context when decrypting because the encryption context is included in the encrypted message that the SDK returns\. But, before it returns the plaintext message, your decrypt function should verify that all pairs in the provided encryption context appear in the encryption context that was used to decrypt the message\.  
First, get a read\-only pointer to the hash table in the session\. This hash table contains the encryption context that was used to decrypt the message\.   

```
const struct aws_hash_table *session_enc_ctx = aws_cryptosdk_session_get_enc_ctx_ptr(session);
```
Then, loop through the encryption context in the `my_enc_ctx` hash table that you copied when encrypting\. Verify that each pair in the `my_enc_ctx` hash table that was used to encrypt appears in the `session_enc_ctx` hash table that was used to decrypt\. If any key is missing, or that key has a different value, stop processing and write an error message\.  

```
for (struct aws_hash_iter iter = aws_hash_iter_begin(my_enc_ctx); !aws_hash_iter_done(&iter);
      aws_hash_iter_next(&iter)) {
     struct aws_hash_element *session_enc_ctx_kv_pair;
     aws_hash_table_find(session_enc_ctx, iter.element.key, &session_enc_ctx_kv_pair)

    if (!session_enc_ctx_kv_pair ||
        !aws_string_eq(
            (struct aws_string *)iter.element.value, (struct aws_string *)session_enc_ctx_kv_pair->value)) {
        fprintf(stderr, "Wrong encryption context!\n");
        abort();
    }
}
```

Clean up the session\.  
After you verify the encryption context, you can destroy the session\. As usual, you can reuse the session\. If you need to reconfigure it, use the `aws_cryptosdk_session_reset` method\.  

```
aws_cryptosdk_session_destroy(session);
```