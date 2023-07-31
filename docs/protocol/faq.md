# 常见问题

## 概念

<details>

<summary><b>"secp256k1"和"ed25519"有什么区别？</b></summary>

secp256k1和ed25519都是用于数字签名和密钥生成的流行密码算法，但它们在安全性、性能和与不同系统的兼容性方面存在一些差异。

secp256k1是一种椭圆曲线算法，广泛用于比特币和许多其他加密货币。它提供128位的安全性，对于大多数实际用途来说被认为足够安全。secp256k1相对较快和高效，适用于需要高性能的应用程序。它得到了大多数密码库和软件的广泛支持，适用于跨平台应用程序。

ed25519是一种较新的椭圆曲线算法，提供与secp256k1相同的128位安全性。然而，由于其对某些攻击类型（如[侧信道攻击](https://en.wikipedia.org/wiki/Side-channel_attack)）的抵抗能力，ed25519通常被认为比secp256k1更安全。它也比许多其他椭圆曲线算法更快，包括secp256k1，适用于需要高性能的应用程序。

在兼容性方面，secp256k1在现有系统中得到更广泛的支持，而ed25519的支持较少。然而，ed25519正在变得越来越受欢迎，并且得到了许多密码库和软件的支持。

在选择secp256k1和ed25519之间时，您应该考虑安全性、性能和兼容性方面的具体需求。如果您正在构建一个需要高性能和与现有系统兼容的应用程序，secp256k1可能是一个更好的选择。然而，如果您正在构建一个需要更高级别安全性和性能的应用程序，并且可以牺牲一些兼容性的应用程序，ed25519可能是一个更好的选择。

</details>


<details>

<summary><b>我在哪里可以找到Evmos的Protobuf接口？</b></summary>

前往我们的[Buf](https://buf.build/evmos)。

</details>


# Frequently Asked Questions

## Concepts

<details>

<summary><b>What is the difference between "secp256k1" and "ed25519"?</b></summary>

secp256k1 and ed25519 are both popular cryptographic algorithms used for digital signatures and key generation, but
they have some differences in terms of security, performance, and compatibility with different systems.

secp256k1 is an elliptic curve algorithm that is widely used in Bitcoin and many other cryptocurrencies. It provides
128-bit security, which is considered sufficient for most practical purposes. secp256k1 is relatively fast and
efficient, making it a good choice for applications that require high performance. It is widely supported by most
cryptographic libraries and software, which makes it a good choice for cross-platform applications.

ed25519 is a newer elliptic curve algorithm that provides 128-bit security, similar to secp256k1. However, ed25519 is
generally considered to be more secure than secp256k1, due to its resistance to certain types of attacks such as
[side-channel attacks](https://en.wikipedia.org/wiki/Side-channel_attack). It is also faster than many other elliptic
curve algorithms, including secp256k1, making it a good choice for applications that require high performance.

In terms of compatibility, secp256k1 is more widely supported by existing systems, while ed25519 is less widely
supported. However, ed25519 is gaining popularity, and is supported by many cryptographic libraries and software.

When choosing between secp256k1 and ed25519, you should consider your specific needs in terms of security, performance,
and compatibility. If you are building an application that requires high performance and compatibility with existing
systems, secp256k1 may be a better choice. However, if you are building an application that requires a higher level
of security and performance, and you can afford to sacrifice some compatibility, ed25519 may be a better choice.

</details>


<details>

<summary><b>Where can I find the Protobuf interfaces for Evmos?</b></summary>

Head over to our [Buf](https://buf.build/evmos).

</details>
