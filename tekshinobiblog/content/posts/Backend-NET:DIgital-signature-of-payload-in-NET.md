---
title: "Backend c#:Reading RSA key pair from PEM files in .NET with C# using Bouncy Castle and Digitally Sign and Verify payload"
date: 2018-09-07T16:09:23+03:00
draft: false 
categories: ["backend"]
tags: ["c#"]
---

.NET does not have an easy way to directly deal with .pem format files generated using OpenSSL. I had to look into Bouncy Castle library to do it. Lets see how. We will also generate a dummy payload and then sign it using the generated pem keys and then verify it.

First let us generate RSA key pair using OpenSSL. Please install OpenSSL before hand.

```shell
[user@host secure]~ openssl genrsa -out posvendor.key.pem 2048
[user@host secure]~ openssl rsa -in posvendor.key.pem -pubout -out posvendor.pub.pem
```
writing RSA key
```shell
[user@host secure]~ ls
```
output:
```shell
posvendor.key.pem
posvendor.pub.pem
```
OK.. so we have public and private keys generated.
For the same of example, here is the path of the pem files generated above:

```cs
string public_pem = "D:\\Projects\\Crypt\\ConsoleApplication1\\posvendor.pub.pem";
string private_pem = "D:\\Projects\\Crypt\\ConsoleApplication1\\posvendor.key.pem";
```

Now install Bouncy castle through nouget.
Also install Newtonsoft Json.

The implementation looks like so:
My includes look like so:
```cs
using Newtonsoft.Json;
using Org.BouncyCastle.Crypto;
using Org.BouncyCastle.Crypto.Parameters;
using Org.BouncyCastle.OpenSsl;
using Org.BouncyCastle.Security;
using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Net;
using System.Security.Cryptography;
using System.Text;
using System.Xml.Serialization;
```
Note that classes `GetPayload`, `GetUnixTime`, `GetNonce` are there to provide some dummy data to form a payload that can be signed and then verified with the above pem keys. Also, `PublicKeyString` and `PrivateKeyString` methods are there to generate a print friendly version of the parsed pem keys.

```cs
public class RsaEnc
    {
        
        private static RSACryptoServiceProvider csp = new RSACryptoServiceProvider(2048);
        private RSAParameters _privateKey;
        private RSAParameters _publicKey;

        public RsaEnc()
        {
            string public_pem = "D:\\Projects\\Crypt\\ConsoleApplication1\\posvendor.pub.pem";
            string private_pem = "D:\\Projects\\Crypt\\ConsoleApplication1\\posvendor.key.pem";


            var pub = RsaEnc.GetPublicKeyFromPemFile(public_pem);
            var pri = RsaEnc.GetPrivateKeyFromPemFile(private_pem);

            
            _publicKey = pub.ExportParameters(false);
            _privateKey = pri.ExportParameters(true);
        }

        public static RSACryptoServiceProvider GetPrivateKeyFromPemFile(string filePath)
        {
            using (TextReader privateKeyTextReader = new StringReader(File.ReadAllText(filePath)))
            {
                AsymmetricCipherKeyPair readKeyPair = (AsymmetricCipherKeyPair)new PemReader(privateKeyTextReader).ReadObject();

                RSAParameters rsaParams = DotNetUtilities.ToRSAParameters((RsaPrivateCrtKeyParameters)readKeyPair.Private);
                RSACryptoServiceProvider csp = new RSACryptoServiceProvider();// cspParams);
                csp.ImportParameters(rsaParams);
                return csp;
            }
        }

        public static RSACryptoServiceProvider GetPublicKeyFromPemFile(String filePath)
        {
            using (TextReader publicKeyTextReader = new StringReader(File.ReadAllText(filePath)))
            {
                RsaKeyParameters publicKeyParam = (RsaKeyParameters)new PemReader(publicKeyTextReader).ReadObject();

                RSAParameters rsaParams = DotNetUtilities.ToRSAParameters((RsaKeyParameters)publicKeyParam);

                RSACryptoServiceProvider csp = new RSACryptoServiceProvider();// cspParams);
                csp.ImportParameters(rsaParams);
                return csp;
            }
        }

        public string PublicKeyString()
        {
            var sw = new StringWriter();
            var xs = new XmlSerializer(typeof(RSAParameters));
            xs.Serialize(sw, _publicKey);
            return sw.ToString();
        }

        public string PrivateKeyString()
        {
            var sw = new StringWriter();
            var xs = new XmlSerializer(typeof(RSAParameters));
            xs.Serialize(sw, _privateKey);
            return sw.ToString();
        }

        public string GetPayload()
        {
            Dictionary<string, string> message = new Dictionary<string, string>
                                                {
                                                    {"0", "0200"},
                                                    {"1", "a238408000c080000000010001000000"},
                                                    {"3", "000000"},
                                                    {"7", "1231235959"},
                                                    {"11", "000135"},
                                                    {"12", "235959"},
                                                    {"13", "1231"},
                                                    {"18", "5814"},
                                                    {"25", "24"},
                                                    {"41", "2222"},
                                                    {"42", "111111111111"},
                                                    {"49", "SGD"},
                                                    {"88", "000000001235"},
                                                    {"104", "demo transaction order_5cb68edb5b9c4"}
                                                };

            string json = JsonConvert.SerializeObject(message, Formatting.None);
            byte[] bytes = Encoding.Default.GetBytes(json);
            string json_utf8encoded = Encoding.UTF8.GetString(bytes);
            return json_utf8encoded;
        }

        public byte[] SignData(byte[] hashOfDataToSign)
        {
            using (var rsa = new RSACryptoServiceProvider(2048))
            {
                rsa.ImportParameters(_privateKey);
                var rsaFormatter = new RSAPKCS1SignatureFormatter(rsa);
                rsaFormatter.SetHashAlgorithm("SHA256");
                return rsaFormatter.CreateSignature(hashOfDataToSign);
            }
        }

        // 10 digit unix time
        public string GetUnixTime()
        {
            var epoch_10_digit_unix_time = (DateTime.Now.ToUniversalTime().Ticks - 621355968000000000) / 10000000;
            return epoch_10_digit_unix_time.ToString();
        }

        public Guid GetNonce()
        {
            Guid guid = Guid.NewGuid();
            return guid;
        }

        public byte[] GetHash(string plaintext)
        {
            HashAlgorithm algorithm = SHA256.Create();
            return algorithm.ComputeHash(Encoding.UTF8.GetBytes(plaintext));
        }

        public bool VerifySignature(byte[] hashOfDataToSign, byte[] signature)
        {
            using (var rsa = new RSACryptoServiceProvider(2048))
            {
                rsa.ImportParameters(_publicKey);
                var rsaDeformatter = new RSAPKCS1SignatureDeformatter(rsa);
                rsaDeformatter.SetHashAlgorithm("SHA256");
                return rsaDeformatter.VerifySignature(hashOfDataToSign, signature);
            }
        }
   }
```
Now you can easily test the implementation. Note that in the testing, I am using private key to sign and public key to verify from the above generated pem keys. In reality, you will use your private key to sign and a public key from someone else to verify (from same person who gave you the payload).

```cs
static void Main(string[] args)
        {
            RsaEnc rs = new RsaEnc();
            
            Console.WriteLine("PublicKey:\n" + rs.PublicKeyString() + "\n");
            Console.WriteLine("PrivateKey:\n" + rs.PrivateKeyString() + "\n");
            Console.WriteLine("Guid:" + rs.GetNonce());
            Console.WriteLine("Epoch:" + rs.GetUnixTime());
            Console.WriteLine("Payload:" + rs.GetPayload() + "\n");
            string for_digesting = rs.GetPayload() + rs.GetUnixTime() + rs.GetNonce();
            byte[] bytes_in_for_digesting = Encoding.UTF8.GetBytes(for_digesting);
            Console.WriteLine("bytes_in_for_digesting:" + bytes_in_for_digesting.Count());
            byte[] for_digesting_hashed = rs.GetHash(for_digesting);
            var sign = rs.SignData(for_digesting_hashed);
            string sign_str = System.Convert.ToBase64String(sign);
            var verify = rs.VerifySignature(for_digesting_hashed, System.Convert.FromBase64String(sign_str));
            HeaderFields fields = new HeaderFields();
            fields.Timestamp = rs.GetUnixTime();
            fields.Nonce = rs.GetNonce().ToString();
            fields.Sign = sign_str;
        }
```
__PS__: the above code snippets are from my experimentation project. This is NOT production code. I have not included exception handling anywhere. Variable assignments and Console.Writelines are all over the place. Please don’t judge. This is more of a scratchpad note for myself, in case I need to revisit the same scenario.