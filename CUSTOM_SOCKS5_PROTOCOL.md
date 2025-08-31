# Custom SOCKS5 Protocol Implementation

This implementation adds support for a custom SOCKS5-based protocol with XOR obfuscation and two new authentication methods (0x80 and 0x82).

## Overview

The custom protocol is based on standard SOCKS5 but includes the following modifications:

1. **XOR Obfuscation**: All client-sent data is XORed with 0xFF
2. **Custom Authentication Methods**: Two new private authentication methods (0x80 and 0x82)
3. **Challenge-Response Authentication**: Both custom methods use challenge-response with HMAC-SHA256

## Protocol Details

### XOR Obfuscation

All data sent by the client is obfuscated by XORing each byte with 0xFF. Server responses are sent without obfuscation.

Example:
- Original: `05 01 80` (SOCKS5 version, 1 method, method 0x80)
- Obfuscated: `FA FE 7F` (after XOR with 0xFF)

### Authentication Method 0x80

**Challenge Format**: Server responds with 1-byte challenge after method selection.

**Authentication Packet** (54 bytes total):
```
+-----+------+----------+------+----------+
| VER | ULEN |  UNAME   | HLEN |   HMAC   |
+-----+------+----------+------+----------+
|  1  |  1   |    19    |  1   |    32    |
+-----+------+----------+------+----------+
```

- **VER**: Version (0x01)
- **ULEN**: Username length (19 bytes exactly)
- **UNAME**: Username (19 bytes)
- **HLEN**: HMAC length (0x20 = 32 bytes)
- **HMAC**: HMAC-SHA256 signature

**HMAC Computation**:
- **Key**: USERNAME + PASSWORD (concatenated)
- **Message**: Single challenge byte from server
- **Algorithm**: HMAC-SHA256

**Handshake Flow**:
1. Client → Server: `05 01 80` (XOR obfuscated: `FA FE 7F`)
2. Server → Client: `05 [challenge_byte]`
3. Client → Server: 54-byte auth packet (XOR obfuscated)
4. Server → Client: `01 00` (success) or other (failure)

### Authentication Method 0x82

**Challenge Format**: Server responds with 4-byte challenge after method selection.

**Authentication Packet** (75 bytes total):
```
+-----+------+----------+------+----------+----------+
| VER | ULEN |  UNAME   | HLEN |   HMAC   | FIXDATA  |
+-----+------+----------+------+----------+----------+
|  1  |  1   |    19    |  1   |    32    |    21    |
+-----+------+----------+------+----------+----------+
```

- **VER**: Version (0x01)
- **ULEN**: Username length (19 bytes exactly)
- **UNAME**: Username (19 bytes)
- **HLEN**: HMAC length (0x20 = 32 bytes)
- **HMAC**: HMAC-SHA256 signature
- **FIXDATA**: Fixed 21-byte data

**HMAC Computation**:
- **Key**: USERNAME + MD5(PASSWORD) as hex string (concatenated)
- **Message**: 4-byte challenge from server
- **Algorithm**: HMAC-SHA256

**Fixed Data** (21 bytes):
```
14 01 01 01 02 04 00 00 00 00 03 02 27 10 04 01 01 05 02 00 04
```

**Handshake Flow**:
1. Client → Server: `05 01 82` (XOR obfuscated: `FA FE 7D`)
2. Server → Client: `05 82 [4_byte_challenge]`
3. Client → Server: 75-byte auth packet (XOR obfuscated)
4. Server → Client: `01 00` (success) or other (failure)

## Configuration

Add the following to your configuration file:

```yaml
socks5:
  port: 1080
  address: 127.0.0.1
  username: 'user_19_bytes_exact'  # Must be exactly 19 bytes
  password: 'your_password'
  auth-method: '0x80'  # or '0x82' or 'standard'
```

### Auth Method Options:
- `standard`: Regular SOCKS5 authentication (default)
- `0x80`: Custom auth with single-byte challenge
- `0x82`: Custom auth with 4-byte challenge and fixed data

## Usage Examples

### Method 0x80 Configuration:
```yaml
socks5:
  port: 1080
  address: 127.0.0.1
  username: 'test_user_12345678'  # Exactly 19 bytes
  password: 'secret_password'
  auth-method: '0x80'
```

### Method 0x82 Configuration:
```yaml
socks5:
  port: 1080
  address: 127.0.0.1
  username: 'test_user_12345678'  # Exactly 19 bytes
  password: 'secret_password'
  auth-method: '0x82'
```

## Important Notes

1. **Username Length**: Both custom methods require usernames to be exactly 19 bytes
2. **Pipeline Mode**: Custom authentication methods disable pipeline mode due to challenge requirements
3. **Obfuscation**: Only client-sent data is obfuscated; server responses use standard format
4. **Backward Compatibility**: Standard SOCKS5 authentication remains fully supported

## Dependencies

- OpenSSL (libcrypto, libssl) for HMAC-SHA256 and MD5 operations

## Building

The implementation automatically includes OpenSSL dependencies:

```bash
make clean
make exec
```

## Testing

A test suite is available to verify the protocol implementation:

```bash
# Compile and run tests
cd /tmp
gcc -o test_custom_socks5 test_custom_socks5.c -lcrypto
./test_custom_socks5
```

This validates:
- XOR obfuscation correctness
- HMAC-SHA256 computation for both auth methods
- MD5 hash computation for method 0x82