客户端代码：
```cpp
#include <iostream>
#include <string>
#include <winsock2.h>
#include <thread>
#pragma comment(lib, "ws2_32.lib")

using namespace std;

const int BUFFER_SIZE = 1024;
const int PORT = 12345;

SOCKET clientSocket;
bool receiving = true;

void receiveMessages() {
    char buffer[BUFFER_SIZE];
    while (receiving) {
        int bytesReceived = recv(clientSocket, buffer, BUFFER_SIZE, 0);
        if (bytesReceived <= 0) {
            cout << "与服务器断开连接" << endl;
            receiving = false;
            break;
        }
        
        string msg(buffer, bytesReceived);
        
        // 检查是否是观战模式提示
        if (msg.find("你已死亡，进入观战模式") != string::npos) {
            cout << "\n=== 你已进入观战模式 ===\n";
            cout << msg << endl;
            cout << "=== 输入继续观察游戏 ===\n";
        } else {
            cout << msg << endl;
        }
    }
}

int main() {
    WSADATA wsaData;
    if (WSAStartup(MAKEWORD(2, 2), &wsaData) != 0) {
        cerr << "WSAStartup failed" << endl;
        return 1;
    }
    
    clientSocket = socket(AF_INET, SOCK_STREAM, 0);
    if (clientSocket == INVALID_SOCKET) {
        cerr << "Socket creation failed" << endl;
        WSACleanup();
        return 1;
    }
    cout<<"目前支持角色：\n狼人\n村民\n预言家\n女巫\n猎人\n守卫\n\n";
    system("pause");
    system("cls");
    string serverIP;
    cout << "输入服务器IP地址: ";
    cin >> serverIP;
    cin.ignore();
    
    sockaddr_in serverAddr;
    serverAddr.sin_family = AF_INET;
    serverAddr.sin_addr.s_addr = inet_addr(serverIP.c_str());
    serverAddr.sin_port = htons(PORT);
    
    if (connect(clientSocket, (sockaddr*)&serverAddr, sizeof(serverAddr)) == SOCKET_ERROR) {
        cerr << "连接服务器失败" << endl;
        closesocket(clientSocket);
        WSACleanup();
        return 1;
    }
    
    string playerName;
    cout << "请输入你的名字: ";
    getline(cin, playerName);
    send(clientSocket, playerName.c_str(), playerName.size() + 1, 0);
    
    // 创建线程接收消息
    thread receiverThread(receiveMessages);
    
    // 主线程处理用户输入
    string input;
    while (receiving) {
        getline(cin, input);
        if (!input.empty()) {
            send(clientSocket, input.c_str(), input.size() + 1, 0);
        }
    }
    
    receiverThread.join();
    closesocket(clientSocket);
    WSACleanup();
    return 0;
}
```

服务端代码：

```
#include <iostream>
#include <vector>
#include <string>
#include <map>
#include <winsock2.h>
#include <time.h>
#include <algorithm>
#include <unordered_set>
#pragma comment(lib, "ws2_32.lib")

using namespace std;

const int MAX_PLAYERS = 12;
const int PORT = 12345;
const int BUFFER_SIZE = 1024;
int minPlayers = 4;  // 默认最少4人
int maxPlayers = 12; // 默认最多12人
bool customSettings = false; // 是否使用自定义设置

enum Role { WEREWOLF, VILLAGER, SEER, WITCH, HUNTER, GUARDIAN };
enum GameState { WAITING, NIGHT_GUARD, NIGHT_WEREWOLF, NIGHT_SEER, NIGHT_WITCH, DAY_DISCUSS, DAY_VOTE, GAME_OVER };

struct Player {
    SOCKET socket;
    string name;
    Role role;
    bool alive;
    bool protectedByGuardian;
    bool poisoned;
    bool saved;
    int id;
};

vector<Player> players;
map<SOCKET, int> socketToPlayerId;
int dayCount = 0;
GameState gameState = WAITING;
int killedPlayerId = -1;
int lastProtectedPlayerId = -1;
bool witchAntidoteUsed = false;
bool witchPoisonUsed = false;
unordered_set<int> werewolfVotes;
unordered_set<int> dayVotes;
void handleNightWerewolf(); 
void handleNightSeer();
void handleNightWitch();
void handleDay();
void handleDayVote();
string roleToString(Role role) {
    switch (role) {
        case WEREWOLF: return "狼人";
        case VILLAGER: return "村民";
        case SEER: return "预言家";
        case WITCH: return "女巫";
        case HUNTER: return "猎人";
        case GUARDIAN: return "守卫";
        default: return "未知";
    }
}

void broadcast(const string& message, bool includeDead = false) {
    char buffer[BUFFER_SIZE];
    strncpy(buffer, message.c_str(), BUFFER_SIZE - 1);
    buffer[BUFFER_SIZE - 1] = '\0';
    
    for (const auto& player : players) {
        if (player.alive || includeDead) {  // 活着的玩家或者观战模式
            send(player.socket, buffer, strlen(buffer) + 1, 0);
        }
    }
    cout << "广播: " << message << endl;
}

void sendToPlayer(int playerId, const string& message) {
    if (playerId < 0 || playerId >= players.size()) return;
    
    char buffer[BUFFER_SIZE];
    strncpy(buffer, message.c_str(), BUFFER_SIZE - 1);
    buffer[BUFFER_SIZE - 1] = '\0';
    
    send(players[playerId].socket, buffer, strlen(buffer) + 1, 0);
    cout << "发送给 " << players[playerId].name << ": " << message << endl;
}

void sendToRole(Role role, const string& message) {
    for (const auto& player : players) {
        if (player.role == role && player.alive) {
            sendToPlayer(player.id, message);
        }
    }
}

void assignRoles() {
    vector<Role> roles;
    
    // 根据玩家人数分配角色
    int werewolfCount = players.size() / 3;
    if (werewolfCount < 1) werewolfCount = 1;
    
    // 添加狼人
    for (int i = 0; i < werewolfCount; ++i) {
        roles.push_back(WEREWOLF);
    }
    
    // 添加特殊角色
    if (players.size() >= 4) {
        roles.push_back(SEER);
        roles.push_back(WITCH);
    }
    if (players.size() >= 6) {
        roles.push_back(HUNTER);
    }
    if (players.size() >= 8) {
        roles.push_back(GUARDIAN);
    }
    
    // 剩余为村民
    while (roles.size() < players.size()) {
        roles.push_back(VILLAGER);
    }
    
    // 随机分配角色
    srand(time(NULL));
    random_shuffle(roles.begin(), roles.end());
    
    for (int i = 0; i < players.size(); ++i) {
        players[i].role = roles[i];
        players[i].alive = true;
        players[i].protectedByGuardian = false;
        players[i].poisoned = false;
        players[i].saved = false;
        players[i].id = i;
        
        // 发送角色信息给玩家
        string msg = "你的身份是: " + roleToString(players[i].role);
        sendToPlayer(i, msg);
        
        // 发送玩家列表
        string playerList = "玩家列表:\n";
        for (int j = 0; j < players.size(); ++j) {
            playerList += to_string(j) + ". " + players[j].name + "\n";
        }
        sendToPlayer(i, playerList);
    }
}

void checkGameOver() {
    int werewolfCount = 0;
    int goodCount = 0;
    
    for (const auto& player : players) {
        if (!player.alive) continue;
        
        if (player.role == WEREWOLF) {
            werewolfCount++;
        } else {
            goodCount++;
        }
    }
    
    if (werewolfCount == 0) {
        broadcast("好人阵营获胜！游戏结束", true);  // 包括死亡玩家
        gameState = GAME_OVER;
        
        // 向所有玩家公布身份
        string roleInfo = "玩家身份揭秘:\n";
        for (const auto& player : players) {
            roleInfo += player.name + ": " + roleToString(player.role) + "\n";
        }
        broadcast(roleInfo, true);
        
    } else if (werewolfCount >= goodCount) {
        broadcast("狼人阵营获胜！游戏结束", true);  // 包括死亡玩家
        gameState = GAME_OVER;
        
        // 向所有玩家公布身份
        string roleInfo = "玩家身份揭秘:\n";
        for (const auto& player : players) {
            roleInfo += player.name + ": " + roleToString(player.role) + "\n";
        }
        broadcast(roleInfo, true);
    }
}

void executePlayer(int playerId) {
    if (playerId < 0 || playerId >= players.size()) return;
    
    players[playerId].alive = false;
    broadcast(players[playerId].name + " 被放逐出局", true);  // 包括死亡玩家
    
    // 通知被放逐的玩家进入观战模式
    string spectatorMsg = "你已死亡，进入观战模式。你可以观察游戏进程但不能参与。\n";
    spectatorMsg += "当前存活玩家:\n";
    for (const auto& player : players) {
        if (player.alive) {
            spectatorMsg += player.name + "\n";
        }
    }
    sendToPlayer(playerId, spectatorMsg);
    
    // 猎人技能
    if (players[playerId].role == HUNTER && players[playerId].alive) {
        sendToPlayer(playerId, "你被放逐，请选择一名玩家带走(输入编号):");
        
        char buffer[BUFFER_SIZE];
        int bytesReceived = recv(players[playerId].socket, buffer, BUFFER_SIZE, 0);
        if (bytesReceived > 0) {
            int targetId = atoi(buffer);
            executePlayer(targetId);
            broadcast(players[playerId].name + " 带走了 " + players[targetId].name, true);
        }
    }
    
    checkGameOver();
}

void handleNightGuard() {
    gameState = NIGHT_GUARD;
    broadcast("天黑请闭眼...");
    
    // 守卫行动
    sendToRole(GUARDIAN, "守卫请睁眼，请选择要守护的玩家(编号):");
    
    // 等待守卫选择
    for (const auto& player : players) {
        if (player.role == GUARDIAN && player.alive) {
            char buffer[BUFFER_SIZE];
            int bytesReceived = recv(player.socket, buffer, BUFFER_SIZE, 0);
            if (bytesReceived > 0) {
                int protectedId = atoi(buffer);
                if (protectedId >= 0 && protectedId < players.size() && protectedId != lastProtectedPlayerId) {
                    players[protectedId].protectedByGuardian = true;
                    lastProtectedPlayerId = protectedId;
                    sendToPlayer(player.id, "你守护了 " + players[protectedId].name);
                } else {
                    sendToPlayer(player.id, "无效的选择或不能连续两晚守护同一人");
                }
            }
        }
    }
    
    Sleep(2000);
    handleNightWerewolf();
}

void handleNightWerewolf() {
    gameState = NIGHT_WEREWOLF;
    werewolfVotes.clear();
    killedPlayerId = -1;
    
    // 狼人行动
    sendToRole(WEREWOLF, "狼人请睁眼，请商量并选择要杀害的玩家(编号):");
    
    // 收集狼人投票
    for (const auto& player : players) {
        if (player.role == WEREWOLF && player.alive) {
            char buffer[BUFFER_SIZE];
            int bytesReceived = recv(player.socket, buffer, BUFFER_SIZE, 0);
            if (bytesReceived > 0) {
                int vote = atoi(buffer);
                werewolfVotes.insert(vote);
                sendToPlayer(player.id, "你选择了杀害 " + players[vote].name);
            }
        }
    }
    
    // 统计狼人投票结果
    map<int, int> voteCount;
    for (int vote : werewolfVotes) {
        voteCount[vote]++;
    }
    
    int maxVotes = 0;
    for (const auto& vc : voteCount) {
        if (vc.second > maxVotes) {
            maxVotes = vc.second;
            killedPlayerId = vc.first;
        }
    }
    
    Sleep(2000);
    handleNightSeer();
}

void handleNightSeer() {
    gameState = NIGHT_SEER;
    
    // 预言家行动
    sendToRole(SEER, "预言家请睁眼，请选择要查验的玩家(编号):");
    
    // 等待预言家选择
    for (const auto& player : players) {
        if (player.role == SEER && player.alive) {
            char buffer[BUFFER_SIZE];
            int bytesReceived = recv(player.socket, buffer, BUFFER_SIZE, 0);
            if (bytesReceived > 0) {
                int checkId = atoi(buffer);
                string result = (players[checkId].role == WEREWOLF) ? "狼人" : "好人";
                sendToPlayer(player.id, players[checkId].name + " 的身份是: " + result);
            }
        }
    }
    
    Sleep(2000);
    handleNightWitch();
}

void handleNightWitch() {
    gameState = NIGHT_WITCH;
    
    // 女巫行动
    for (const auto& player : players) {
        if (player.role == WITCH && player.alive) {
            // 通知女巫被杀玩家
            if (killedPlayerId != -1 && !players[killedPlayerId].protectedByGuardian) {
                string msg = "今晚被杀的是: " + players[killedPlayerId].name + "\n";
                if (!witchAntidoteUsed) {
                    msg += "是否使用解药? (1-使用, 0-不使用):";
                }
                sendToPlayer(player.id, msg);
                
                if (!witchAntidoteUsed) {
                    char buffer[BUFFER_SIZE];
                    int bytesReceived = recv(player.socket, buffer, BUFFER_SIZE, 0);
                    if (bytesReceived > 0) {
                        int useAntidote = atoi(buffer);
                        if (useAntidote == 1) {
                            players[killedPlayerId].saved = true;
                            witchAntidoteUsed = true;
                            sendToPlayer(player.id, "你使用了解药救了 " + players[killedPlayerId].name);
                            killedPlayerId = -1;
                        }
                    }
                }
                
                if (!witchPoisonUsed) {
                    sendToPlayer(player.id, "是否使用毒药? 如果是，请输入要毒杀的玩家编号，否则输入-1:");
                    char buffer[BUFFER_SIZE];
                    int bytesReceived = recv(player.socket, buffer, BUFFER_SIZE, 0);
                    if (bytesReceived > 0) {
                        int poisonId = atoi(buffer);
                        if (poisonId >= 0 && poisonId < players.size()) {
                            players[poisonId].poisoned = true;
                            witchPoisonUsed = true;
                            sendToPlayer(player.id, "你毒杀了 " + players[poisonId].name);
                        }
                    }
                }
            }
        }
    }
    
    Sleep(2000);
    handleDay();
}

// 修改白天讨论处理
void handleDay() {
    dayCount++;
    gameState = DAY_DISCUSS;
    
    // 重置守卫保护
    for (auto& player : players) {
        player.protectedByGuardian = false;
    }
    
    // 处理死亡玩家
    vector<string> deadPlayers;
    if (killedPlayerId != -1 && !players[killedPlayerId].saved && !players[killedPlayerId].protectedByGuardian) {
        players[killedPlayerId].alive = false;
        deadPlayers.push_back(players[killedPlayerId].name + "(被狼人杀害)");
        
        // 通知被杀的玩家进入观战模式
        string spectatorMsg = "你已死亡，进入观战模式。你可以观察游戏进程但不能参与。\n";
        spectatorMsg += "当前存活玩家:\n";
        for (const auto& player : players) {
            if (player.alive) {
                spectatorMsg += player.name + "\n";
            }
        }
        sendToPlayer(killedPlayerId, spectatorMsg);
    }
    
    // 处理被毒杀的玩家
    for (auto& player : players) {
        if (player.poisoned && player.alive) {
            player.alive = false;
            deadPlayers.push_back(player.name + "(被女巫毒杀)");
            player.poisoned = false;
            
            // 通知被毒的玩家进入观战模式
            string spectatorMsg = "你已死亡，进入观战模式。你可以观察游戏进程但不能参与。\n";
            spectatorMsg += "当前存活玩家:\n";
            for (const auto& p : players) {
                if (p.alive) {
                    spectatorMsg += p.name + "\n";
                }
            }
            sendToPlayer(player.id, spectatorMsg);
        }
    }
    
    // 公布死亡信息 (包括观战玩家)
    if (deadPlayers.empty()) {
        broadcast("天亮了，今天是第 " + to_string(dayCount) + " 天\n昨晚是平安夜，没有人死亡", true);
    } else {
        string msg = "天亮了，今天是第 " + to_string(dayCount) + " 天\n昨晚死亡的玩家有:\n";
        for (const auto& name : deadPlayers) {
            msg += name + "\n";
        }
        broadcast(msg, true);
    }
    
    // 检查游戏是否结束
    checkGameOver();
    if (gameState == GAME_OVER) return;
    
    // 讨论环节 (只有活着的玩家可以参与)
    broadcast("请开始讨论... (有BUG，请输入/vote进入投票阶段)");
    
    // 等待讨论结束
    bool voting = false;
    while (!voting) {
        for (const auto& player : players) {
            if (player.alive) {  // 只有活着的玩家可以讨论
                char buffer[BUFFER_SIZE];
                int bytesReceived = recv(player.socket, buffer, BUFFER_SIZE, 0);
                if (bytesReceived > 0) {
                    string msg(buffer);
                    if (msg == "/vote") {
                        voting = true;
                        break;
                    } else {
                        broadcast(player.name + ": " + msg);
                    }
                }
            }
        }
    }
    
    handleDayVote();
}


// 修改投票处理函数
void handleDayVote() {
    gameState = DAY_VOTE;
    dayVotes.clear();
    
    broadcast("讨论结束，开始投票。请选择要放逐的玩家(编号):");
    
    // 收集投票 (只有活着的玩家可以投票)
    for (const auto& player : players) {
        if (player.alive) {
            char buffer[BUFFER_SIZE];
            int bytesReceived = recv(player.socket, buffer, BUFFER_SIZE, 0);
            if (bytesReceived > 0) {
                int vote = atoi(buffer);
                if (vote >= 0 && vote < players.size() && players[vote].alive) {
                    dayVotes.insert(vote);
                    sendToPlayer(player.id, "你投票给了 " + players[vote].name);
                } else {
                    sendToPlayer(player.id, "无效的投票");
                }
            }
        }
    }
    
    // 统计投票结果
    map<int, int> voteCount;
    for (int vote : dayVotes) {
        voteCount[vote]++;
    }
    
    int maxVotes = 0;
    int executeId = -1;
    for (const auto& vc : voteCount) {
        if (vc.second > maxVotes) {
            maxVotes = vc.second;
            executeId = vc.first;
        }
    }
    
    if (executeId != -1) {
        executePlayer(executeId);
    } else {
        broadcast("投票没有结果，无人被放逐", true);  // 包括观战玩家
    }
    
    if (gameState != GAME_OVER) {
        Sleep(3000);
        handleNightGuard();
    }
}
void clientHandler(SOCKET clientSocket) {
    char buffer[BUFFER_SIZE];
    int bytesReceived;
    
    // 接收玩家名称
    bytesReceived = recv(clientSocket, buffer, BUFFER_SIZE, 0);
    if (bytesReceived <= 0) {
        closesocket(clientSocket);
        return;
    }
    
    string playerName(buffer);
    Player newPlayer;
    newPlayer.socket = clientSocket;
    newPlayer.name = playerName;
    newPlayer.alive = true;
    newPlayer.id = players.size();
    players.push_back(newPlayer);
    socketToPlayerId[clientSocket] = newPlayer.id;
    
    cout << playerName << " 加入了游戏 (当前玩家: " << players.size() << "/" << maxPlayers << ")" << endl;
    broadcast(playerName + " 加入了游戏 (当前玩家: " + to_string(players.size()) + "/" + to_string(maxPlayers) + ")");
    
    if (players.size() >= minPlayers && gameState == WAITING) {
        // 询问是否开始游戏
        broadcast("已达到最低人数要求(" + to_string(minPlayers) + "人)，是否开始游戏？(输入/start开始，/wait继续等待)");
        
        // 等待管理员决定
        bool decisionMade = false;
        while (!decisionMade) {
            for (const auto& player : players) {
                bytesReceived = recv(player.socket, buffer, BUFFER_SIZE, 0);
                if (bytesReceived > 0) {
                    string cmd(buffer);
                    if (cmd == "/start") {
                        decisionMade = true;
                        gameState = NIGHT_GUARD;
                        broadcast("游戏开始! 玩家数量: " + to_string(players.size()));
                        assignRoles();
                        handleNightGuard();
                        break;
                    } else if (cmd == "/wait") {
                        decisionMade = true;
                        broadcast("继续等待更多玩家加入... (当前 " + to_string(players.size()) + "/" + to_string(maxPlayers) + ")");
                        break;
                    }
                }
            }
        }
    }
    
    // 检查是否达到最大人数
    if (players.size() >= maxPlayers && gameState == WAITING) {
        broadcast("已达到最大玩家数量(" + to_string(maxPlayers) + ")，游戏即将开始");
        Sleep(2000);
        gameState = NIGHT_GUARD;
        broadcast("游戏开始! 玩家数量: " + to_string(players.size()));
        assignRoles();
        handleNightGuard();
    }
}
// 添加服务器设置函数
void serverSettings() {
    cout << "当前设置: 最少 " << minPlayers << " 人，最多 " << maxPlayers << " 人" << endl;
    cout << "是否使用默认设置? (y/n): ";
    char choice;
    cin >> choice;
    
    if (choice == 'n' || choice == 'N') {
        customSettings = true;
        cout << "请输入最少玩家数量(4-12): ";
        cin >> minPlayers;
        minPlayers = max(4, min(12, minPlayers));
        
        cout << "请输入最多玩家数量(" << minPlayers << "-12): ";
        cin >> maxPlayers;
        maxPlayers = max(minPlayers, min(12, maxPlayers));
        
        cout << "设置完成: 最少 " << minPlayers << " 人，最多 " << maxPlayers << " 人" << endl;
    }
}

int main() {
	    // 显示欢迎信息
    cout << "=== 狼人杀游戏服务器 ===" << endl;
    cout << "1. 启动服务器" << endl;
    cout << "2. 服务器设置" << endl;
    cout << "选择: ";
    
    int choice;
    cin >> choice;
    cin.ignore();
    
    if (choice == 2) {
        serverSettings();
    }
    WSADATA wsaData;
    if (WSAStartup(MAKEWORD(2, 2), &wsaData) != 0) {
        cerr << "WSAStartup failed" << endl;
        return 1;
    }
    
    SOCKET serverSocket = socket(AF_INET, SOCK_STREAM, 0);
    if (serverSocket == INVALID_SOCKET) {
        cerr << "Socket creation failed" << endl;
        WSACleanup();
        return 1;
    }
    
    sockaddr_in serverAddr;
    serverAddr.sin_family = AF_INET;
    serverAddr.sin_addr.s_addr = INADDR_ANY;
    serverAddr.sin_port = htons(PORT);
    
    if (bind(serverSocket, (sockaddr*)&serverAddr, sizeof(serverAddr)) == SOCKET_ERROR) {
        cerr << "Bind failed" << endl;
        closesocket(serverSocket);
        WSACleanup();
        return 1;
    }
    
    if (listen(serverSocket, MAX_PLAYERS) == SOCKET_ERROR) {
        cerr << "Listen failed" << endl;
        closesocket(serverSocket);
        WSACleanup();
        return 1;
    }
    
    cout << "狼人杀服务器已启动，等待玩家连接..." << endl;
    
    while (true) {
        SOCKET clientSocket = accept(serverSocket, NULL, NULL);
        if (clientSocket == INVALID_SOCKET) {
            cerr << "Accept failed" << endl;
            continue;
        }
        
        // 创建线程处理客户端
        CreateThread(NULL, 0, (LPTHREAD_START_ROUTINE)clientHandler, (LPVOID)clientSocket, 0, NULL);
    }
    
    closesocket(serverSocket);
    WSACleanup();
    return 0;
}
```
