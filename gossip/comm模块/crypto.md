crypto
===

crypto文件中定义了与证书有关的方法

# writeFile方法

将证书数据写入文件

参数：

- filename string
- keyType string
- data []byte

返回值

- error

```golang
f, err := os.Create(filename)
if err != nil {
	return err
}
defer f.Close()
return pem.Encode(f, &pem.Block{Type: keyType, Bytes: data})
```

# GenerateCertificatesOrPanic方法

产生证书

首先，调用ecdsa.GenerateKey(elliptic.P256(), rand.Reader)方法生成一对公私钥；

调用x509.CreateCertificate方法生成证书

```golang
sn, err := rand.Int(rand.Reader, new(big.Int).Lsh(big.NewInt(1), 128))
if err != nil {
	panic(err)
}
template := x509.Certificate{
	KeyUsage: x509.KeyUsageKeyEncipherment | x509.KeyUsageDigitalSignature,
	SerialNumber: sn,
	ExtKeyUsage: []x509.ExtKeyUsage{x509.ExtKeyUsageServerAuth},
}
rawBytes, err := x509.CreateCertificate(rand.Reader, &template, &template, &privateKey.PublicKey, privateKey)
```

将证书和私钥写入文件，并调用tls.LoadX509KeyPair生成tls证书

```golang
err = writeFile(certKeyFile, "CERTIFICATE", rawBytes)
...
privBytes, err := x509.MarshalECPrivateKey(privateKey)
...
err = writeFile(privBytes, "EC PRIVATE KEY", privBytes)
...
cert, err := tls.LoadX509KeyPair(certKeyFile, privKeyFile)
...
if len(cert.Certificate) == 0 {
	panic(errors.New("Certificate chain is empty"))
}
return cert
```

# certHashFromRawCert方法

计算cert的SHA256哈希

```golang
if len(rawCert) == 0 {
	return nil
}
return util.ComputeSHA256(rawCert)
```

# extractCertificateHashFromContext方法

从stream中抽取证书的hash

```golang
pr, extracted := peer.FromContext(ctx)
...
authInfo := pr.AuthInfo
...
tlsInfo, isTLSConn := authInfo.(credentials.TLSInfo)
...
certs := tlsInfo.State.PeerCertificates
...
raw := certs[0].Raw
return certHashFromRawCert(raw)
```