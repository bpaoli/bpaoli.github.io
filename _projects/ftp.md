---
name: File Transfer Protocol
tools: [C++, Socket Programming, Unix]
image: file.jpg
description: A custom-built protocol project designed for reliable file transfer between clients and servers over unpredictable networks with built in testing for real-world scenarios.
---

# Simple Reliable File Transfer Protocol

## Overview

Simple reliable file transfer is a set of two programs used to send files over a network from a client to a server. The client reads from a file, splits it into packets, and sends them to the server. The server reads the packets and stores them at a specified path. A custom transfer protocol ensures reliability, and the client software is threaded to handle multiple servers simultaneously. It is designed to work with the Unix socket API.

### Features

- **Testing Features**: Includes options to simulate packet drop rates, adjust window sizes, and set the Maximum Transmission Unit (MTU) size.
- **Logging**: Packets are logged on both the client and server sides for tracking.
- **Threading**: Client can handle multiple servers concurrently.

---

## The Transfer Protocol

1. **Server Initialization**:  
   The server starts by creating and binding a port, waiting for a SYN packet from a client.

2. **Client Initialization**:  
   - Reads a configuration file to determine servers to replicate to.
   - Opens the input file and calculates the maximum sequence number (based on file size and MTU).
   - Sends a SYN packet to the server containing:
     ```
     <maximum sequence number> <window size> <MTU> <output path>
     ```

3. **Handshake**:  
   - The server responds with an ACK packet (sequence number 0).  
   - If the client doesn’t receive an ACK, it retries the SYN up to 5 times before closing with an error.

4. **Data Transfer**:  
   - **Protocol**: Based on a Go-Back-N sliding window protocol.
   - **Server**: Processes packets, writes them to the file, and manages a buffer for out-of-order packets.  
   - **Client**: Sends packets and waits for ACKs before advancing the base sequence number.  

5. **Error Handling**:  
   - Duplicate ACKs improve performance by preventing unnecessary retransmissions.  
   - Server closes connections after 150 seconds of inactivity.

---

## Internal Server Design

- **Initialization**:  
  - Validates arguments with regex patterns.  
  - Creates and binds a socket using `socketbind()`.  

- **Session Management**:  
  - Uses `sessionstart()` to wait for connections.  
  - On receiving a SYN, forks a child process to handle file transfer.

- **Helper Functions**:  
  - **`ack()`**: Sends ACKs.  
  - **`rcv()`**: Receives packets, writes sequential packets to file, and manages a buffer for out-of-order packets.  
  - **`log()`**: Logs packet events with RFC 3339 formatting.  
  - **`createDir()`**: Creates output directories as needed.  

---

## Internal Client Design

- **Initialization**:  
  - Validates arguments with regex patterns.  
  - Calculates the maximum sequence number using `getSize()`.  

- **Flow Control**:  
  - Manages file transfer with `flowcontrol()`:
    - Handles SYN/ACK handshake.  
    - Sends packets in chunks defined by the window size.  
    - Waits for ACKs and adjusts the base sequence number.  

- **Helper Functions**:  
  - **`syn()`**: Formats and sends SYN packets.  
  - **`ackWait()`**: Waits for ACK packets with a 30-second timeout.  
  - **`snd()`**: Reads the input file and sends packets.  
  - **`log()`**: Logs packet events similarly to the server.  

---

## Shortcomings

1. **SYN Packet Limitation**:  
   - Minimum MTU size is limited by the SYN packet size (minimum 16 bytes).  
   - Smaller MTU sizes reduce the maximum file size that can be transferred.

2. **No FIN Packet**:  
   - Late ACKs can cause the client to resend final packets unnecessarily.  

3. **Final ACK Drop**:  
   - If the server’s final ACK is dropped, the client may time out even if the file transfer is complete.


## Client Side Code
```c++
#include <sys/socket.h>
#include <cstring>
#include <regex>
#include <unistd.h>
#include <iostream>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <fstream>
#include <cstring>
#include <signal.h>
#include <filesystem>
#include <cmath>
#include <set>


void log(bool data, bool dropdata, bool ack, bool dropack, int seq, int baseseq, int win){
    auto now = std::chrono::system_clock::now();
    std::time_t now_c = std::chrono::system_clock::to_time_t(now);
    std::tm* utc_tm = std::gmtime(&now_c);
    char buffer[30];
    std::strftime(buffer, sizeof(buffer), "%Y-%m-%dT%H:%M:%S", utc_tm);
    auto milliseconds = std::chrono::duration_cast<std::chrono::milliseconds>(now.time_since_epoch()) % 1000;
    if(data){
        std::cout << buffer << "." << std::setfill('0') << std::setw(3) << milliseconds.count() << "Z" <<", DATA, " << seq <<  ", " << baseseq << ", " << (seq+1) << ", " << baseseq + win << std::endl;
    }else if(dropdata){
        std::cout << buffer << "." << std::setfill('0') << std::setw(3) << milliseconds.count() << "Z" <<", DROP DATA, " << seq <<  ", " << baseseq << ", " << (seq+1) << ", " << baseseq + win << std::endl;
    }else if(ack){
        std::cout << buffer << "." << std::setfill('0') << std::setw(3) << milliseconds.count() << "Z" <<", ACK, " << seq <<  ", " << baseseq << ", " << (seq+1) << ", " << baseseq + win << std::endl;
    }else if(dropack){
        std::cout << buffer << "." << std::setfill('0') << std::setw(3) << milliseconds.count() << "Z" <<", DROP ACK, " << seq <<  ", " << baseseq << ", " << (seq+1) << ", " << baseseq + win << std::endl;
    }
}


int getSize(std::ifstream& inputFile, int mtu){
    int maxseq;
    std::streampos fileSize;
    inputFile.seekg(0, std::ios::end);
    if((fileSize = inputFile.tellg()) == -1){
        std::cerr << "Error determining file size" << std::endl;
        std::exit(EXIT_FAILURE);
    }
    inputFile.seekg(0, std::ios::beg);
    int fileSizeInt = static_cast<int>(fileSize);
    if((maxseq = std::ceil(fileSizeInt / (mtu-5))) == 0){
        return maxseq + 1;
    }
    return maxseq + 1;
}


//This function is used to create the socket and set the connection to check for errors
int socketBind(sockaddr_in& serverAddr, int port, std::string ip) {
    int socketID;
    bzero(&serverAddr, sizeof(serverAddr)); //Zeros out the struct memory
    serverAddr.sin_family = AF_INET; //Set the struct parameters
    serverAddr.sin_addr.s_addr = htonl(INADDR_ANY);
    serverAddr.sin_port = htons(port);
    if (inet_pton(AF_INET, ip.c_str(), &serverAddr.sin_addr) <= 0) { //Convert the struct to binary
        std::cerr << "inet_pton error" << std::endl;
        std::exit(EXIT_FAILURE);
    }
    if ((socketID = socket(AF_INET, SOCK_DGRAM, 0)) < 0) { // Create the socket
        std::cerr << "Error creating socket" << std::endl;
        std::exit(EXIT_FAILURE);
    }
    if (connect(socketID, (struct sockaddr*)&serverAddr, sizeof(serverAddr)) < 0) { // Run connect to check for errors
        std::cerr << "connect() returned an error" << std::endl;
        std::exit(EXIT_FAILURE);
    }
    return socketID;
}


void syn(int socketID, int mtu, std::string outFilePath, int maxseq, int win) {
    ssize_t p;
    std::string syndta = std::to_string(maxseq) + " " + std::to_string(win) + " " + std::to_string(mtu) + " " + outFilePath;
    char sendbuf[300];
    // Check if syndta fits into sendbuf
    if (syndta.size() >= sizeof(sendbuf)) {
        std::cerr << "syndta exceeds the size of sendbuf" << std::endl;
        return; // Exit the function
    }
    strncpy(sendbuf, syndta.c_str(), sizeof(sendbuf) - 1);
    sendbuf[sizeof(sendbuf) - 1] = '\0';
    //log(false, false, true, false, 0, 0, 0);
    std::cout << "ACK " << 0 << std::endl;
    if ((p = write(socketID, sendbuf, 300)) < 0) {
        if (errno == EINTR) {
            std::cerr << "Cannot detect server" << std::endl;
        } else {
            std::cerr << "Write error" << std::endl;
        }
        std::exit(EXIT_FAILURE);
    }
}


int ackWait(int socketID){
    char recbuf[6]; //Server response buffer
    int n, seq;
    fd_set readfds;
    FD_ZERO(&readfds);
    FD_SET(socketID, &readfds);
    struct timeval timeout;
    timeout.tv_sec = 3; // 30 seconds timeout
    timeout.tv_usec = 0;
    int ready = select(socketID + 1, &readfds, nullptr, nullptr, &timeout);
    if(ready == -1){//Select error
        std::cerr << "select() returned an error" << std::endl;
        std::exit(EXIT_FAILURE);
    }else if(ready == 0){//Timeout
        return -1;
    }else{
        if ((n = read(socketID, recbuf, sizeof(recbuf))) < 0) { //Read the response from the server
            std::cerr << "Cannot detect server" << std::endl;
            return -1;
        }
    }
    std::regex pattern(R"(^(\d*))");
    std::smatch match;
    std::string seqStr(recbuf);
    if(std::regex_search(seqStr, match, pattern)){
        seq = stoi(match[1].str());
    }
    std::cout << "ACK " << seq << std::endl;
    return seq;
}


void snd(int socketID, int mtu, int curseq, std::streamoff position, int win, std::string inFilePath) {
    std::ifstream inputFile(inFilePath, std::ios::binary);
    int p, baseseq;
    inputFile.seekg(position, std::ios::beg);
    baseseq = curseq;
    for(int i = 0; i < win; i++) {//Loop window size number of times
        char readbuf[mtu-5]; //Send and recieve message buffers
        std::string seqstr = std::to_string(curseq);
        inputFile.read(readbuf, sizeof(readbuf));
        readbuf[inputFile.gcount()] = '\0'; //Read the data for the current packet
        std::string readstr(readbuf);
        std::string sendbuf = std::to_string(curseq) + " " + readstr;
        //log(true, false, false, false, curseq, baseseq, win);
        std::cout << "Data " << curseq << std::endl;
        if ((p = write(socketID, sendbuf.c_str(), sendbuf.size())) < 0) { //Send the packet to the server
            if(errno == EINTR) {
                std::cerr << "Cannot detect server" << std::endl;
                std::exit(EXIT_FAILURE);
            }else{
                std::cerr << "Write error" << std::endl;
                std::exit(EXIT_FAILURE);
            }
        }
        curseq += 1;
    }
    inputFile.close();
}


int main(int argc, char *argv[]) {
    /////////////////////////////////////INPUT VERIFICATION///////////////////////////////////////////////////////////
    if (argc != 7) { //Check for the correct number of arguments
        std::cerr << "Incorrect number of arguments" << std::endl;
        std::exit(EXIT_FAILURE);
    }
    std::string ip = argv[1];  
    std::string portStr = argv[2];
    std::string mtuStr = argv[3];
    std::string winStr = argv[4];
    std::string inFilePath = argv[5];
    std::string outFilePath = argv[6];
    std::string in = ip + " " + portStr + " " + mtuStr + " "+ winStr +" " + inFilePath + " " + outFilePath;
    int port, mtu, win;
    std::regex pattern(R"(^((25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)(\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)){3}) (\d{1,5}) (\d*) (\d*) (.*) (.*)$)");//Regex pattern to verify input
    std::smatch match;
    if (!(std::regex_search(in, match, pattern))) { // Verify the input
        std::cerr << "Incorrect input format" << std::endl;
        std::exit(EXIT_FAILURE);
    } else {
        port = std::stoi(argv[2]);
        mtu = std::stoi(argv[3]);
        win = std::stoi(argv[4]);
    }
    if(mtu < 6){//Verify the correct mtu size
        std::cerr << "Minimum MTU is 6." << std::endl;
        std::exit(EXIT_FAILURE);
    }
    if(win < 1){//Verify the correct mtu size
        std::cerr << "Minimum window size is 1." << std::endl;
        std::exit(EXIT_FAILURE);
    }


    ////////////////////////////////////////CREATE SOCKET///////////////////////////////////////////////
    int socketID;
    struct sockaddr_in serverAddr;
    socketID = socketBind(serverAddr, port, ip); //Create the socket and connect it
    std::ifstream inputFile(inFilePath, std::ios::binary);
    if (!(inputFile.is_open())) {// Check if both files opened successfully
        std::cerr << "Error opening the input file" << std::endl;
        std::exit(EXIT_FAILURE);
    }
    int maxseq = getSize(inputFile, mtu);
    int synack = -1;
    int ackCount = 0;


    ///////////////////////////////////////////////SEND SYN////////////////////////////////////////////////////////
    while(synack == -1){
        if(ackCount > 0){
            //std::cerr << "Packet loss detected" << std::endl;
        }
        if(ackCount == 5){ //Syn was attempted 5 times with 5 failues. Server assumed to be unreachable
            std::cerr << "Reached max retransmission limit" << std::endl;
            std::exit(EXIT_FAILURE);
        }
        syn(socketID, mtu, outFilePath, maxseq, win); //send the syn packet with all of the necessary connection parameters to the server
        synack = ackWait(socketID); //wait for the syn to be acked
        ackCount += 1; //Increment the number of times the packet was sent
    }
    inputFile.close();
    ////////////////////////////////////////////START TRANSFER//////////////////////////////////
    int curseq = 0;
    int prevseq = 0;
    std::streamoff byteOffset = 0;
    int resendcounter = 0;
    int tempwin = win;
    while(curseq < maxseq){//Repeat until file is completely sent
        if(maxseq < (win + curseq)){ //adjust the window size since we are on the last few packets
            tempwin = maxseq-curseq;
        }
        snd(socketID, mtu, curseq + 1, byteOffset, tempwin, inFilePath); //send tempwin number of packets
        int ackseq;
        int counter = 0;
        std::set<int> ack;
        while(counter < tempwin){//need to collect all of the acked seq in one shot in case of unordered acks
            if((ackseq = ackWait(socketID)) == -1){
                //std::cerr << "Packet loss detected" << std::endl;
                break;
            }
            //std::cout << "Ack recieved: " << ackseq << std::endl;
            ack.insert(ackseq);
            counter += 1;
        }
        for (int num : ack) {
            if(num == (curseq +1)){
                curseq++;
            }
        }
        if(curseq == prevseq){
            resendcounter += 1;
        }else{
            resendcounter = 0;
        }
        if(resendcounter == 5){
            std::cerr << "Reached max retransmission limit" << std::endl;
            std::exit(EXIT_FAILURE);
        }
        byteOffset = (curseq * (mtu-5));//move the pointer the the first byte past the last acked position
        prevseq = curseq;
    }
    std::cout << "Transmission complete" << std::endl;
    exit(0);
}
```














# Server Side Code
```c++
#include <sys/socket.h>
#include <cstring>
#include <regex>
#include <unistd.h>
#include <iostream>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <fstream>
#include <cstring>
#include <signal.h>
#include <filesystem>
#include <map>
#include <random>
#include <sys/wait.h>
﻿
void log(bool data, bool dropdata, bool ack, bool dropack, int seq){
    auto now = std::chrono::system_clock::now();
    std::time_t now_c = std::chrono::system_clock::to_time_t(now);
    std::tm* utc_tm = std::gmtime(&now_c);
    char buffer[30];
    std::strftime(buffer, sizeof(buffer), "%Y-%m-%dT%H:%M:%S", utc_tm);
    auto milliseconds = std::chrono::duration_cast<std::chrono::milliseconds>(now.time_since_epoch()) % 1000;
    if(data){
        std::cout << buffer << "." << std::setfill('0') << std::setw(3) << milliseconds.count() << "Z" <<", DATA, " << seq << std::endl;
    }else if(dropdata){
        std::cout << buffer << "." << std::setfill('0') << std::setw(3) << milliseconds.count() << "Z" <<", DROP DATA, " << seq << std::endl;
    }else if(ack){
        std::cout << buffer << "." << std::setfill('0') << std::setw(3) << milliseconds.count() << "Z" <<", ACK, " << seq << std::endl;
    }else if(dropack){
        std::cout << buffer << "." << std::setfill('0') << std::setw(3) << milliseconds.count() << "Z" <<", DROP ACK, " << seq << std::endl;
    }
}
﻿
void sig_alrm(int) {
    std::cerr << "Connection with client lost" << std::endl;
    std::exit(EXIT_FAILURE);
    return;
}
﻿
//Creates a directory at the specified output path if it doesn't already exist
void createDir(std::string outFilePath){
    std::string directoryPath;
    std::regex pattern(R"(^(.*/)?(.*)$)"); //Regex pattern to split the path and file
    std::smatch match;
    if (std::regex_search(outFilePath, match, pattern)) {
        directoryPath = match[1];
        std::filesystem::path outDir = directoryPath;
        if (!directoryPath.empty() && (!std::filesystem::exists(outDir))) { //Check if the directory already exists
            std::filesystem::create_directories(outDir);
        }
        std::string textFile = match[2];
    }
    return;
}
﻿
void ack(int seq, int socketID, struct sockaddr* clientAddr, socklen_t len){
    char buf[6];
    std::string ack = std::to_string(seq);
    strncpy(buf, ack.c_str(), sizeof(buf) - 1);
    buf[sizeof(buf) - 1] = '\0';
    log(false, false, true, false, seq);
    //std::cout <<"Acked: " <<buf << std::endl;
    int bytesSent = sendto(socketID, buf, 10, 0, clientAddr, len); //Send the ack
    if (bytesSent < 0) { //Catch any errors
        std::cerr << "Sendto error: " << strerror(errno) << std::endl;
        std::exit(EXIT_FAILURE);
    }
}
﻿
void rcv(int socketID, int maxseq, size_t win, int mtu, std::ofstream& outputFile, struct sockaddr* clientAddr, socklen_t size, int perc){
    signal(SIGALRM, sig_alrm);
    std::regex pattern(R"((\d+) ((.|\r|\n)*))");
    std::smatch match;
    int curseq = 0;
    std::random_device rd;
    std::mt19937 gen(rd());
    std::uniform_int_distribution<int> dis(1, 100);
    std::map<int, std::string> window;
    while(curseq < maxseq){    
        int seq;
        std::string message;
        char buf[mtu + 1];
        alarm(150);
        ssize_t nbytes = recvfrom(socketID, buf, sizeof(buf), 0, clientAddr, &size);
        if (nbytes < 0) {
            std::cerr << "Error receiving data" << std::endl;
            std::exit(EXIT_FAILURE);
        }
        buf[nbytes] = '\0';
        std::string data(buf);
        if(std::regex_search(data, match, pattern)){
            seq = stoi(match[1].str());
            message = match[2].str();
            //std::cout << seq << std::endl;
        }        
        int rand = dis(gen);
        if(rand > perc){ //Packet not dropped
            log(true, false, false, false, seq);
            if(seq == curseq + 1){
                outputFile.write(message.c_str(), message.size());
                ack(curseq + 1, socketID, clientAddr, size);
                curseq++;
                while (window.find(curseq + 1) != window.end()) {
                    outputFile.write(window[curseq + 1].c_str(), window[curseq + 1].size());
                    window.erase(curseq + 1);
                    ack(curseq + 1, socketID, clientAddr, size);
                    curseq++;
                }
            }else if(window.count(seq) == 0){
                ack(seq, socketID, clientAddr, size);
                window[seq] = message;
            }
﻿
        }else{//packet dropped
            log(false, true, false, false, seq);
        }
        alarm(0);
    }
}
﻿
//This function recieves messages sequentially and then echos them back to the sender
void sessionStart(int socketID, struct sockaddr* clientAddr, socklen_t size, int perc) {
    int n;
    while (1) { //Loop infinitely to keep the server running
        //Recieve the sync packet
        char buf[300];
        memset(buf, 0, sizeof(buf));
        std::cout << "Waiting for transmission request" << std::endl;
        if((n = recvfrom(socketID, buf, sizeof(buf), 0, clientAddr, &size)) > 0){
            //fork
            pid_t pid = fork();
            if (pid == -1) {
                std::cerr << "Fork failed" << std::endl;
                continue;
            }else if(pid == 0){
                buf[sizeof(buf) - 1] = '\0';
                int maxseq, win, mtu;
                std::string outFilePath;
                std::regex pattern(R"(^(\d*) (\d*) (\d*) (.*)$)");
                std::smatch match;
                std::string buff(buf);
                if(std::regex_search(buff, match, pattern)){
                    maxseq = stoi(match[1].str());
                    win = stoi(match[2].str());
                    mtu = stoi(match[3].str());
                    outFilePath = match[4].str();
                    createDir(outFilePath);
                    std::ofstream outputFile(outFilePath, std::ios::binary);
                    std::random_device rd;
                    std::mt19937 gen(rd());
                    std::uniform_int_distribution<int> dis(1, 100);
                    int rand = dis(gen);
                    if(rand > perc){//Syn not dropped
                        log(false, false, true, false, 0);        
                        ack(0, socketID, clientAddr, size);//if this ack drops system fails
                        rcv(socketID, maxseq, win, mtu, outputFile, clientAddr, size, perc);
                        outputFile.close();
                        std::cout << "Transmission complete" << std::endl;
                        _exit(EXIT_SUCCESS);
                    }else{//Syn is dropped
                        log(false, false, false, true, 0);
                        std::cerr << "Synchrinization failure" << std::endl;
                        outputFile.close();
                        exit(EXIT_FAILURE);
                    }
                }
            } else {
                // Parent process
                int status;
                waitpid(pid, &status, 0);  // Wait for child process to finish
            }
        }
    }
}
﻿
//This function is used to create the socket and bind it
int socketBind(sockaddr_in serverAddr, int port) {
    int socketID = socket(AF_INET, SOCK_DGRAM, 0); //Create the socket
    if (socketID < 0) { //Check if the socket was created successfully
        perror("socket() failed");
        std::exit(EXIT_FAILURE);
    }
    bzero(&serverAddr, sizeof(serverAddr)); //Zero out the input struct
    serverAddr.sin_family = AF_INET; //Set the socket parameters
    serverAddr.sin_addr.s_addr = htonl(INADDR_ANY);
    serverAddr.sin_port = htons(port);
﻿
    if (bind(socketID, (struct sockaddr*)&serverAddr, sizeof(serverAddr)) < 0) { //Bind the socket
        perror("bind() failed");
        std::exit(EXIT_FAILURE);
    }
﻿
    return socketID;
}
﻿
int main(int argc, char* argv[]) {
    if (argc != 3) { // Check for the correct number of arguments
        std::cerr << "Incorrect number of arguments" << std::endl;
        std::exit(EXIT_FAILURE);
    }
    int port, perc;
    std::string portStr = argv[1];
    std::string percStr = argv[2];
    std::string input = portStr + " "+percStr;
    std::regex pattern(R"(^(\d{1,5}) (0|100|[1-9]\d?)$)");//Regex to check against the input
    std::smatch match;
    if (!(std::regex_search(input, match, pattern))) { //Input verification
        std::cerr << "Incorrect input format" << std::endl;
        std::exit(EXIT_FAILURE);
    } else {
        port = std::stoi(argv[1]); //Convert the input to a number
        perc = std::stoi(argv[2]);
    }
    int socketID;
    struct sockaddr_in serverAddr, cliaddr;
    socketID = socketBind(serverAddr, port); //Create the socket and bind it
    sessionStart(socketID, (struct sockaddr*)&cliaddr, sizeof(cliaddr), perc); //Start the server functionality
    return 0;
}
```