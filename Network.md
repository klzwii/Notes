TLS 1.2 vs TLS 1.3
1.2:
认证需要2RTT  
- Client->Server  
Hello CipherSuit ClientRandom CompressSuit Time SessionID VersionSupported
- Server->Client  
Hello CipherMethod ServerRandom SessionID Time Version  
Certificate(May from CA) *  
ServerKeyExchange(RSA/DH) *  
CertificateRequest*  
ServerHello Done  
- Client->Server  
Certificate *  
ClientKeyExchange  
CertificateVerify*  
ChangeCipher(encrypted by master key)  
Finished  
- Server->Client
ChangeCipher(encrypted by master key)  
Finished  

1.3:
认证只需要1RTT
不再支持RSA 只支持(EC)DHE
- Client->Server  
key_share：对全部椭圆曲线及有限域算出对应的公钥
- Server->Client
key_share：所选出的椭圆曲线/有限域及对应的公钥
