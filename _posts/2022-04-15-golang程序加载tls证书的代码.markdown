```go'func loadTLSConfig(c *ServiceConfig) *tls.Config {
	cfg := &tls.Config{}

	if len(c.CertFile) > 0 && len(c.KeyFile) > 0 {
		cert, err := tls.LoadX509KeyPair(c.CertFile, c.KeyFile) // "example-cert.pem" "example-key.pem
		if err != nil {
			log.Printf("Failed to load cert file or key file %v %v, error: %+v\n", c.CertFile, c.KeyFile, err)
			return nil
		}
		cfg.Certificates = []tls.Certificate{cert}
	}

	if len(c.RootCasFile) > 0 {
		certPool := x509.NewCertPool()
		rootBuf, err := os.ReadFile(c.RootCasFile) // " root_ca.pem"
		if err != nil {
			log.Printf("failed load rootCasFile %v, error: %+v", err)
			return nil
		}
		if !certPool.AppendCertsFromPEM(rootBuf) {
			log.Printf("failed to append rootCasFile, error: %+v", err)
			return nil
		}
		tls.RootCas = certPool
	}

	if len(c.ServerName) > 0 {
		cfg.ServerName = c.ServerName // serverName需要与服务器证书内的Common Name一致
	}

	cfg.InsecureSkipVerify = c.InsecureSkipVerify
	return cfg
}
```
