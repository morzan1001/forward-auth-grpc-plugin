# GRPC Forward Auth Plugin

## Overview

The GRPC Forward Auth Plugin for Traefik allows validating incoming requests against a gRPC authentication service. It forwards authentication requests to the gRPC authentication service and checks the response to decide whether to allow or deny the request.

## Installation

To install the plugin, add the following configuration to your Traefik configuration file:

```yaml
pilot:
  token: "your_pilot_token"

experimental:
  plugins:
    grpcForwardAuth:
      moduleName: "github.com/morzan1001/forward-auth-grpc-plugin"
      version: "v0.1.0"
```

## Configuration

The plugin is configured via the `traefik.yml` file. Here is an example:

```yaml
http:
  middlewares:
    my-grpc-forward-auth:
      plugin:
        grpcForwardAuth:
          address: "localhost:50051"
          tokenHeader: "authorization"
```

### Parameters

- `address`: The address of the gRPC authentication service.
- `tokenHeader`: The name of the header that contains the authentication token.

### Exampe gRPC Auth Service configuration

The gRPC authentication service must implement the following endpoints to be compatible with the plugin:

#### proto/auth.proto

```
syntax = "proto3";

package auth;

option go_package = "github.com/morzan1001/forward-auth-grpc-plugin/proto";

// AuthService handles authentication requests
service AuthService {
    rpc Authenticate(AuthRequest) returns (AuthResponse);
}

// AuthRequest contains the authentication token
message AuthRequest {
    string token = 1;
}

// AuthResponse contains the authentication result
message AuthResponse {
    bool allowed = 1;
    string message = 2;
    map<string, string> metadata = 3;
}
```

#### AuthService Implementation

```go
package main

import (
    "context"
    "net"

    pb "github.com/morzan1001/forward-auth-grpc-plugin/proto"
    "google.golang.org/grpc"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)

type AuthServiceServer struct {
    pb.UnimplementedAuthServiceServer
}

func (s *AuthServiceServer) Authenticate(ctx context.Context, req *pb.AuthRequest) (*pb.AuthResponse, error) {
    // Implement your authentication logic here
    if req.Token == "valid-token" {
        return &pb.AuthResponse{
            Allowed: true,
            Message: "Authentication successful",
        }, nil
    }
    return &pb.AuthResponse{
        Allowed: false,
        Message: "Authentication failed",
    }, status.Error(codes.Unauthenticated, "invalid token")
}

func main() {
    lis, err := net.Listen("tcp", ":50051")
    if err != nil {
        panic(err)
    }
    grpcServer := grpc.NewServer()
    pb.RegisterAuthServiceServer(grpcServer, &AuthServiceServer{})
    if err := grpcServer.Serve(lis); err != nil {
        panic(err)
    }
}
```

### Tests

To test the plugin, you can use the provided unit tests. Run the tests with the following command:

```bash
go test ./...
```

or 

```bash
make test
```

### License

This project is licensed under the MIT License. See the LICENSE file for more details.