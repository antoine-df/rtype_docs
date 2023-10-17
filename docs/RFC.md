# Network Protocol for Game Client-Server Communication

**Status:** Informational  
**Created by:** Matthis, Jacques, Antoine, Louis  
**Version:** 1.0  
**Date:** October 10, 2023

## Abstract

This document presents a client-server communication protocol for the game "R-Type," utilizing a combination of UDP and TCP. The protocol facilitates the exchange of structured data between clients and the server through a custom serializer/deserializer.

## 1. Introduction

The primary objective of this protocol is to streamline communication between a video game client and server while employing a custom serializer/deserializer. It is versatile, supporting both UDP for low-latency communication and TCP for reliability.

## 2. Message Header

All messages exchanged between the client and server adhere to a common structure. The message header encompasses the following elements:
- **Message Type:** Binary
- **Message Size:** 64 bits
- **Session Identifier:** Connection ID

## 3. UDP Communication

### 3.1. Advantages and Limitations

UDP-based communication is reserved for low-latency operations, including real-time player position updates.

### 3.2. UDP Message Format

UDP messages conform to the common message header structure, followed by data specific to the message type.

### 3.3. Server-Side UDP Data Reception and Deserialization

The server-side UDP data reception and deserialization process involves listening for incoming data from connected clients. The server uses a `while` loop to manage incoming data over the UDP protocol.

#### 3.3.1. Data Reception via UDP

The server continuously listens for incoming data from UDP clients using the following code:

```cpp
while (this->_run) {
    int bytes_received = recvfrom(this->_socket, buffer, BUFFER_STREAM_UDP, 0, (struct sockaddr *)&from, (socklen_t *)&from_length);

    if (bytes_received == SOCKET_ERROR) {
        #ifdef _WIN32
            printf("recvfrom() failed with error %d\n", WSAGetLastError());
        #else
            perror("recvfrom() failed");
        #endif
        return;
    }

    // ...
}
```

#### 3.3.2. Data Deserialization for UDP

The received data is then processed and deserialized using the `ReaderStream` and `Serializer` classes, similar to the TCP deserialization process. Key deserialization steps are as follows:

```cpp
oengine::opack::Serializer serializer;
_stream->allocation_reader_stream(this->_buffer, this->_bytes_received);

if (_stream->_completed) {
    serializer.data(_stream->data());
    _request_name = serializer >> _request_name;
    _id = serializer >> _id;
    //printf("Request Name: %s, Id: %d\n", request_name.c_str(), id);
    serializer.clear();
    _stream->clear();
}
```

### 4. TCP Communication

#### 4.1. Advantages and Limitations

TCP-based communication is employed for operations necessitating reliability, such as authentication and transactions.

#### 4.2. TCP Message Format

TCP messages adhere to the common message header structure, followed by data specific to the message type. TCP communication guarantees the preservation of message order.

### 4.3. Server-Side TCP Data Reception and Deserialization

The server-side TCP data reception and deserialization process involves listening for incoming data from connected clients. The server uses a `while` loop to manage incoming data over the TCP protocol.

#### 4.3.1. Data Reception via TCP

The server continuously receives data via a TCP connection using the following code:

```cpp
while (this->connected) {
    try {
        if ((r = recv(this->_csocket, buffer, BUFFER_STREAM_TCP, 0)) == 0) {
            this->close_client();
            break;
        }
        _stream->allocation_reader_stream(buffer, r);
        _stream->dump();
        if (_stream->_completed) 
        {
            // Deserialize the data
            serializer.data(_stream->data());
            request_name = serializer >> request_name;
            id = serializer >> id;
            printf("Request Name: %s, Id: %d\n", request_name.c_str(), id);

            // Further processing of the deserialized data can be performed here

            serializer.clear();
            _stream->clear();
        }
    }
    catch (std::exception e) {
        this->close_client();
    }
}
```

#### 4.3.2. Data Deserialization for TCP

The received data is then processed and deserialized using the `ReaderStream` and `Serializer` classes. Key deserialization steps are as follows:

```cpp
// Deserialize the data
serializer.data(_stream->data());
request_name = serializer >> request_name;
id = serializer >> id;

// Further processing of the deserialized data

 can be performed here
```

## 5. Serializer/Deserializer

The custom serializer/deserializer, which is utilized to convert data to bytes and vice versa, is elaborated upon in the associated source code.

## 6. Security

The protocol includes a connection-level authentication system for UDP communication.

## 7. Client-Side Data Serialization and Transmission

This section provides an overview of how data is serialized and transmitted from the client to the server using both UDP and TCP. Examples of data serialization and transmission are presented for each protocol.

### 7.1. Serialization for UDP:

Data serialization for UDP enables clients to prepare structured data for transmission. The following code snippet illustrates how data can be serialized:

```cpp
std::string requestName("OnConnection");
int id = 10;
oengine::opack::Serializer serializer(requestName.size() + 8);
serializer << requestName;
serializer << id;
oengine::opack::byte *serializedData = serializer.data();
size_t sizeDataSerialized = serializer.count_data(requestData);
```

### 7.2. Sending Serialized Data over UDP:

After data serialization, clients can transmit the serialized data over the UDP protocol to the server. The following code demonstrates this process:

```cpp
int _socketClient = socket(AF_INET, SOCK_DGRAM, 0);
if (_socketClient < 0) {
    perror("socket error");
    return false;
};
memset(&_serverAddr, 0, sizeof(_serverAddr));
_serverAddr.sin_family = AF_INET;
_serverAddr.sin_port = htons(_serverPort);
if (inet_pton(AF_INET, _serverIP.c_str(), &_serverAddr.sin_addr) <= 0) {
    perror("inet_pton error");
    return false;
}
ssize_t bytesSent = sendto(_socketClient, data, size, 0, (struct sockaddr*)&_serverAddr, sizeof(_serverAddr));
```

### 7.3. Serialization for TCP:

For data transmission over TCP, the client must first serialize the data. The code snippet below illustrates how data can be serialized for TCP transmission:

```cpp
std::string requestName("OnConnection");
int id = 10;
oengine::opack::Serializer serializer(requestName.size() + 8);
serializer << requestName;
serializer << id;
oengine::opack::byte *requestData = serializer.data();
size_t sizeData = serializer.count_data(requestData);
```

### 7.4. Sending Serialized Data over TCP:

Once the data is properly serialized, clients establish a TCP connection with the server and transmit the data. The following code demonstrates this process:

```cpp
if ((socketClient = socket(AF_INET, SOCK_STREAM, 0)) == -1) {
    perror("socket error");
    exit(O_EXIT_FAILURE);
}
struct sockaddr_in addressClient;
addressClient.sin_family = AF_INET;
addressClient.sin_port = htons(serverPort);
if (inet_pton(AF_INET, serverIP.c_str(), &addressClient.sin_addr) <= 0) {
    perror("inet_pton error");
    exit(O_EXIT_FAILURE);
}
if (connect(socketClient, (struct sockaddr *)&addressClient, sizeof(addressClient)) < 0) {
    perror("connect error");
    exit(O_EXIT_FAILURE);
}
std::cout << "Connected to the server" << std::endl;
write(socketClient, requestData, sizeData);
```

This client-side data serialization and transmission process is essential for sending structured information to the server, enabling the client to communicate effectively within the game network.

## 8. Conclusion

This client-server communication protocol, which combines UDP and TCP, serves as a robust foundation for communication within a video game environment. Specific details, including message types, supported operations, and security measures, should be tailored to suit the game's unique requirements. The protocol facilitates efficient data exchange between clients and the server, contributing to a seamless gaming experience.
