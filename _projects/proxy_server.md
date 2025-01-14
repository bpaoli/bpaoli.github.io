---
name: C2S Proxy
tools: [C++, Socket Programming, Unix]
image: https://cdn.pixabay.com/photo/2013/07/13/10/17/computer-156950_1280.png
description: A proxy server that converts HTTP GET and HEAD requests to HTTPS, with blacklist management and traffic logging.
---
# C2S Proxy

The C2S proxy is a proxy server that takes HTTP GET and HEAD requests and converts them to HTTPS to send to an external server. It allows sites to be blacklisted and includes traffic logging. It is build on the Unix socket API.

---

## Connection Management and Cleanup

Threads are used for the purpose of concurrency. The proxy first creates a listen port, and a while loop is set to wait for connections. When a request is received from a client, a new thread is created, and a handler function is called to deal with the transmission. The handler function sets a permanent while loop to deal with the persistent connection:

1. **Request Parsing**: The client’s request is parsed using regular expressions. Formatting errors or unsupported operations result in appropriate HTTP error messages, and the proxy closes the connection.
2. **Blacklist Check**: The URL specified by the client is sent to a helper function that checks it against the server blacklist. If clean, DNS resolves its IP address.
3. **Server Connection**: The proxy creates an SSL port and attempts to connect to the server. If this fails, ports are freed, and the client connection is broken. On success, the client’s request is forwarded to the server.
4. **Server Reply**: A loop handles the server’s reply:
   - Empty replies free ports and close the client connection.
   - Valid replies are sent back to the client.
5. **Termination**: The loop repeats until an unsupported request, timeout, or connection closure occurs. Threads then rejoin the master thread.

---

## Internal Proxy Design

### Overview
The `main()` function handles input validation and initial setup:
- **File Validation**: Input files are verified, and missing directories or files are created using `createDir()`.
- **Socket Binding**: A proxy socket is created and bound via `socketBind()`.
- **Session Handling**: Connections and threading are managed by `session()`.

### Functions

#### `createDir()`
Splits the path and filename using regex, checks for directory existence, and creates missing directories or files.

#### `socketBind()`
Creates and binds a socket to a specified port, defining parameters like port number, address, and IPv4.

#### `session()`
Manages threading and connections:
- Defines a signal for `Ctrl+C` functionality.
- Calls `listen()` on the server socket and waits for connections in a loop.
- Spawns new threads for each connection, executing `handler()`.

#### `handler()`
Handles client-server interactions:
- Opens the blacklist file.
- Creates SSL and context structures.
- Checks for `Ctrl+C` to reload the blacklist.
- Parses client HTTP requests:
  - Sends a 400 error for formatting issues or 501 for unsupported operations.
  - Splits the URL and port from HTTP messages, resolving IP addresses and checking the blacklist.
- Handles server connections and responses:
  - Forwards client messages to the server.
  - Reads and processes server responses, logging interactions.

#### `resolveIP()`
Checks input (IP or URL) against the blacklist:
- Resolves IP addresses using `gethostbyaddr()` or `gethostbyname()`.
- Compares results with the blacklist.

#### `search()`
Walks through the blacklist file, checking if IPs or URLs match. Returns 0 for matches, 1 otherwise.

#### `log()`
Logs client-server interactions:
- Retrieves the current date and time.
- Formats and appends interaction details to the log file.

#### `cleanup()`
Closes ports and SSL contexts when a connection ends.

#### `signalHandler()`
Triggered by `Ctrl+C`. Sets a global flag to reload the blacklist file.

#### `reloadfile()`
Closes and reopens the blacklist file to update its contents.

---

## Shortcomings

1. **`Ctrl+C` Functionality**: The program directly accesses the blacklist file. Changes to the file are immediately accessible, but threading instability prevents maintaining a static file image. `Ctrl+C` forces updates but doesn't fully align with the intended design.

2. **Reliance on Regular Expressions**: Slight deviations in HTTP implementations can cause parsing failures despite efforts to ensure robustness.

3. **Complexity of Design**: For robustness, most functionality is centralized in `handler()`. This increases complexity but avoids errors caused by over-modularization.

4. **Optimization**: While the code executes quickly, further optimization is possible.

---

## Code
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


void sig_alrm(int) {
    std::cerr << "Connection with client lost" << std::endl;
    std::exit(EXIT_FAILURE);
    return;
}


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


        }else{//packet dropped
            log(false, true, false, false, seq);
        }
        alarm(0);
    }
}


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


    if (bind(socketID, (struct sockaddr*)&serverAddr, sizeof(serverAddr)) < 0) { //Bind the socket
        perror("bind() failed");
        std::exit(EXIT_FAILURE);
    }


    return socketID;
}


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