这里主要是用 dh-hamc 算法来进行信道安全的验证

dh算法，可以确保双方交换得到一个共享的密钥，但没有提供双方身份验证服务，容易受到中间人攻击。

hmac算法可以提供身份验证，但它的安全性，需要双方使用双方事先约定的密钥，这个dh算法能保证。

所以dh与hmac配合使用，可以确保信道安全。这个算法，传输的是公钥，私钥 secret 是通过交换key算出来的。

#### 1.Server->Client : base64(8bytes random challenge)

	 即服务器发送一个8 bytes challenge 给客户端

服务器代码
```lua
local challenge = crypt.randomkey()
send_pack(peer_sock, crypt.base64encode(challenge))

print(string.format("challenge is %s\n", challenge))
print(string.format("challenge len is %s\n", #challenge))
```

客户端收到这个8 bytes challenge

客户端代码
```lua
local challenge = crypt.base64decode(recv(sock))
print(string.format("challenge  is %s\n", challenge))
print(string.format("challenge len is %s\n", #challenge))
```

#### 2.Client->Server : base64(8bytes handshake client key)

  即客户端发送 clientkey 给服务器

客户端代码
```lua
local clientkey = crypt.randomkey()
send(sock, crypt.base64encode(crypt.dhexchange(clientkey)))

print(string.format("clientkey is %s\n", crypt.base64encode(crypt.dhexchange(clientkey))))
print(string.format("clientkey len is %s\n", #crypt.base64encode(crypt.dhexchange(clientkey))))
```
服务器接收这个clientkey

服务端代码
```lua
local handshake = recv_pack(peer_sock)
local clientkey = crypt.base64decode(handshake)
if #clientkey ~= 8 then
  print("Invalid client key")
  return "Invalid client key"
end
```

#### 3.服务器生成 serverkey 发送给客户端

服务器代码
```lua
local serverkey = crypt.randomkey()
send_pack(peer_sock, crypt.base64encode(crypt.dhexchange(serverkey)))

print(string.format("serverkey is %s\n", crypt.dhexchange(serverkey)))
print(string.format("serverkey len is %s\n", #crypt.dhexchange(serverkey)))
```
客户端读取 serverkey

客户端代码
```lua
local serverkey = crypt.base64decode(recv(sock))

print(string.format("serverkey is %s\n", serverkey))
print(string.format("serverkey len is %s\n", #serverkey))
```
#### 4.客户端服务器根据 clientkey和serverkey 计算secret 即私匙

客户端代码
```lua
local secret = crypt.dhsecret(serverkey, clientkey)
```
服务器代码
```lua
local secret = crypt.dhsecret(clientkey, serverkey)
```
#### 	5. Client->Server : base64(HMAC(challenge, secret))

	即客户端发给服务器hmac值, hmac(challenge, secret)

客户端代码
```lua
local hmac = crypt.hmac64(challenge, secret)
send(sock, crypt.base64encode(hmac))
```

服务器接收这个 hmac 值

服务器代码
```lua
local response = recv_pack(peer_sock)  
```

#### 6.服务器判断安全验证是否成功

服务器根据之前计算好的 secret与challenge 做hmac运算得到自己的hmac值，并且与之前收到的hmac值即response作对比，如果两个值则相等验证成功返回 "security_verify_success"
给客户端端，否则返回 "security_verify_failure".

服务器代码
```lua
local hmac = crypt.hmac64(challenge, secret)

local verify_result = "security_verify_failure"
if hmac == crypt.base64decode(response) then
  verify_result = "security_verify_success"
end

send_pack(peer_sock, crypt.base64encode(verify_result))
```
客户端接收服务器的验证结果

客户端代码
```lua
local verify_result = crypt.base64decode(recv(sock))
print("verify_result: ", verify_result)

if "security_verify_failure" == verify_result then
  sock:close()
end
```

#### 7.客户端 服务器完整代码

客户端代码
```lua
local function security_verify(sock)
	-- 1. Server->Client : base64(8bytes random challenge)
	-- 读取用于握手验证的challenge

	local challenge = crypt.base64decode(recv(sock))
	print(string.format("challenge  is %s\n", challenge))
	print(string.format("challenge len is %s\n", #challenge))

	-- 2. Client->Server : base64(8bytes handshake client key)
	-- 发送 clientkey
	local clientkey = crypt.randomkey()
	send(sock, crypt.base64encode(crypt.dhexchange(clientkey)))

	print(string.format("clientkey is %s\n", crypt.base64encode(crypt.dhexchange(clientkey))))
	print(string.format("clientkey len is %s\n", #crypt.base64encode(crypt.dhexchange(clientkey))))

	-- 4. Server->Client : base64(DH-Exchange(server key))
	-- 读取 serverkey
	local serverkey = crypt.base64decode(recv(sock))

	print(string.format("serverkey is %s\n", serverkey))
	print(string.format("serverkey len is %s\n", #serverkey))

	-- 5. Server Client secret := DH-Secret(client key/server key)
	-- 客户端和服务器各自计算secret, secret = clientkey 和 serverkey 做 dhexchange 的结果，也就是私匙
	local secret = crypt.dhsecret(serverkey, clientkey)

	-- 6. Client->Server : base64(HMAC(challenge, secret))
	-- 客户端发给服务器hmac值, hmac(challenge, secret)
	local hmac = crypt.hmac64(challenge, secret)
	send(sock, crypt.base64encode(hmac))

	print(string.format("hmac is %s\n", hmac))

	-- 7.接收来自服务器的验证结果
	local verify_result = crypt.base64decode(recv(sock))
	print("verify_result: ", verify_result)

	if "security_verify_failure" == verify_result then
		sock:close()
	end
end
```
服务器完整代码

```lua
function tools.security_verify(peer_sock)
	-- 1. Server->Client : base64(8bytes random challenge)
	-- 发送 challenge
	local challenge = crypt.randomkey()
	send_pack(peer_sock, crypt.base64encode(challenge))
	print(string.format("challenge is %s\n", challenge))
	print(string.format("challenge len is %s\n", #challenge))

	-- 2. Client->Server : base64(8bytes handshake client key)
	-- 收到 clientkey
	local handshake = recv_pack(peer_sock)
	local clientkey = crypt.base64decode(handshake)
	if #clientkey ~= 8 then
		print("Invalid client key")
		return "Invalid client key"
	end

	print(string.format("clientkey is %s\n", handshake))
	print(string.format("clientkey len is %s\n", #handshake))

	-- 3. Server: Gen a 8bytes handshake server key
    -- 4. Server->Client : base64(DH-Exchange(server key))
	-- 发送 serverkey
	local serverkey = crypt.randomkey()
	send_pack(peer_sock, crypt.base64encode(crypt.dhexchange(serverkey)))

	print(string.format("serverkey is %s\n", crypt.dhexchange(serverkey)))
	print(string.format("serverkey len is %s\n", #crypt.dhexchange(serverkey)))

	-- 5. Server Client secret := DH-Secret(client key/server key)
	-- 客户端和服务器各自计算secret, secret = clientkey 和 serverkey 做 dhexchange 的结果
	local secret = crypt.dhsecret(clientkey, serverkey)

	-- 6. Client->Server : base64(HMAC(challenge, secret))
	-- 客户端发给服务器hmac值, hmac(challenge, secret)
	-- 收到hamc
	local response = recv_pack(peer_sock)  
	local hmac = crypt.hmac64(challenge, secret)

	print(string.format("response is %s\n", crypt.base64decode(response)))
	print(string.format("hmac   is %s\n",   hmac))

	-- 7.根据收到的response(即客户端发来的hamc)与服务器计算的hmac值昨对比
	local verify_result = "security_verify_failure"
	if hmac == crypt.base64decode(response) then
		verify_result = "security_verify_success"
	end

	send_pack(peer_sock, crypt.base64encode(verify_result))
	return verify_result
end
```
参考：

https://github.com/cloudwu/skynet/wiki/LoginServer
https://github.com/peimin/metoo/blob/master/lualib/snax/login_server.lua#L71=L105
https://github.com/peimin/metoo/blob/master/client.lua
http://snowolf.iteye.com/blog/382422
http://www.lovelucy.info/openssl-dh-key-exchange.html
http://stamen.iteye.com/blog/1558255
https://en.wikipedia.org/wiki/Hash-based_message_authentication_code


