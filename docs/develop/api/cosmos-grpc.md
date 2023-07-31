---
sidebar_position: 3
---

# Cosmos gRPC & REST

## Cosmos gRPC

Evmos为所有集成的Cosmos SDK模块提供了gRPC端点。这使得钱包和区块浏览器可以更轻松地与权益证明逻辑和原生Cosmos交易和查询进行交互。

## Cosmos HTTP REST (gRPC-Gateway)

[gRPC-Gateway](https://grpc-ecosystem.github.io/grpc-gateway/)读取gRPC服务定义并生成一个反向代理服务器，将RESTful JSON API转换为gRPC。使用gRPC-Gateway，用户可以使用REST与Cosmos gRPC服务进行交互。在这里可以查看支持的gRPC-Gateway API端点列表，使用Swagger [here](../api#clients)。


---
sidebar_position: 3
---

# Cosmos gRPC & REST

## Cosmos gRPC

Evmos exposes gRPC endpoints for all the integrated Cosmos SDK modules. This makes it easier for
wallets and block explorers to interact with the Proof-of-Stake logic and native Cosmos transactions and queries.

## Cosmos HTTP REST (gRPC-Gateway)

[gRPC-Gateway](https://grpc-ecosystem.github.io/grpc-gateway/) reads a gRPC service definition and
generates a reverse-proxy server which translates RESTful JSON API into gRPC. With gRPC-Gateway,
users can use REST to interact with the Cosmos gRPC service. See the list
of supported gRPC-Gateway API endpoints using Swagger [here](../api#clients).
