# OpenBuffer
OpenBuffer is used to read, write and parse binary streams, and is designed for network data streams.

**The OpenLinyou project designs a cross-platform server framework. Write code in VS or XCode and run it on Linux without any changes, even on Android and iOS.**
OpenLinyou：https://github.com/openlinyou
https://gitee.com/linyouhappy

## Cross-platform support 
Designed for cross-platforms such as Windows and Linux.

## Compilation and execution
Please install the cmake tool and use it to build the project. It can be compiled and run on VS or Xcode. 
Source code: https://github.com/openlinyou/openbuffer
```
git clone https://github.com/openlinyou/openbuffer
cd ./openbuffer
mkdir build
cd build
cmake ..
# If it is win32, openbuffer.sln will appear in this directory. Click it to start VS for coding and debugging.
make
./test
```

## All source files
+ src/openbuffer.h
+ src/openbuffer.cpp

## Technical Features
The technical features of OpenBuffer: 
1. For adding data push operation, use node linked list to manage memory. Each time the data is pushed, the data is stored on a node. If the capacity of the node is exceeded, a new node will be created for storage.

2. To read data from OpenBuffer, first merge the data on all nodes, store them in continuous memory, and then release the memory of the corresponding node.

3. It supports reading and writing fixed length integers and unfixed length integers.


## 1.Serialization and Deserialization
Supports integer and variable length integer serialization. Adopt small end coding.

```C++
#include <assert.h>
#include <string.h>
#include <string>
#include <vector>
#include "openbuffer.h"

using namespace open;

int main()
{
    OpenBuffer openBuffer;

    char data[256] = "Hello OpenBuffer!";
    const std::string str = "Hello OpenLinyou!";
    
    size_t len = strlen(data);
    openBuffer.pushBack(&len, sizeof(len));
    openBuffer.pushBack(data, len);
 
    len = str.size();
    openBuffer.pushUInt32((int)len);
    openBuffer.pushBack(str.data(), len);

    openBuffer.pushUInt16(1616);
    openBuffer.pushUInt32(3232);
    openBuffer.pushUInt64(6464);

    openBuffer.pushVInt32(0x79);
    openBuffer.pushVInt32(0x80);
    openBuffer.pushVInt32(0x4000);
    openBuffer.pushVInt32(0x10000000);

    openBuffer.pushVInt64(0x10000001);

    std::vector<char> vectData;
    len = 0;
    openBuffer.popFront(&len, sizeof(len));
    vectData.resize(len);
    openBuffer.popFront(vectData.data(), len);
    assert(memcmp(vectData.data(), data, len) == 0);
    
    char ret[256] = {};
    uint32_t len1 = 0;
    openBuffer.popUInt32(len1);
    openBuffer.popFront(ret, len1);
    assert(str == ret);
    
    unsigned short u16 = 0;
    openBuffer.popUInt16(u16);
    assert(u16 == 1616);

    uint32_t u32 = 0;
    openBuffer.popUInt32(u32);
    assert(u32 == 3232);
    uint64_t u64 = 0;
    openBuffer.popUInt64(u64);
    assert(u64 == 6464);

    uint64_t v32 = 0;
    openBuffer.popVInt64(v32);
    assert(v32 == 0x79);

    uint64_t v32_1 = 0;
    openBuffer.popVInt64(v32_1);
    assert(v32_1 == 0x80);

    uint64_t v32_2 = 0;
    openBuffer.popVInt64(v32_2);
    assert(v32_2 == 0x4000);

    uint64_t v32_3 = 0;
    openBuffer.popVInt64(v32_3);
    assert(v32_3 == 0x10000000);

    uint64_t v64 = 0;
    openBuffer.popVInt64(v64);
    assert(v64 == 0x10000001);
    
    return 0;
}
```

## 2. Parsing network data packets
Simulate socket packets and parse http messages. Resolve HTTP packet contamination issues.

```C++
#include <assert.h>
#include <string.h>
#include <string>
#include <vector>
#include "openbuffer.h"

using namespace open;

int main()
{
    std::vector<std::string> datas = {
        "HTTP/1.1 200 OK@&Connection: keep-alive@&Content-Type: application/x-javascript@&",
        "Date: Sat, 18 Mar 2023 08:11:44 GMT@&Strict-Transport-Security: max-age=31536000@&Traceco",
        "de: 24764974122629742602031816@&Vary: Accept-Encoding@&"
    };
    std::string body = "Hello OpenBuffer!!Hello OpenBuffer!!";
    datas.push_back("content-length:" + std::to_string(body.size()) + "@&");
    datas.push_back("@&" + body);
    datas.push_back("@&");
    OpenBuffer openBuffer;
    for (size_t x = 0; x < 10000; x++)
    {
        openBuffer.clear();
        bool isHeader = true;
        size_t k = 0;
        std::string head;
        for (size_t i = 0; i < datas.size(); ++i)
        {
            openBuffer.pushBack(datas[i].data(), datas[i].size());
            if (isHeader)
            {
                unsigned char* tmp = openBuffer.data();
                for (; k < openBuffer.size() - 3; k++)
                {
                    //find @&@&
                    if (tmp[k] == '@' && tmp[k + 1] == '&' && tmp[k + 2] == '@' && tmp[k + 3] == '&')
                        break;
                }
                if (k >= openBuffer.size() - 3) continue;

                k += 4;
                openBuffer.popFront(head, k);
                isHeader = false;
            }
        }
        std::string test = body + "@&";
        std::string buffer;
        buffer.append((const char*)openBuffer.data(), openBuffer.size());
        assert(test == buffer);
    }
    return 0;
}
```

## 3. Slice data 
```C++
#include <assert.h>
#include <string.h>
#include <string>
#include <vector>
#include "openbuffer.h"

using namespace open;

int main()
{
    char data[8] = {1, 2, 3, 4, 5, 6, 7, 8};
    OpenSlice slice((unsigned char*)data, sizeof(data));
    for (size_t i = 1; i < slice.size() + 1; i++)
    {
        char tmp = 0;
        slice.popFront(&tmp, 1);
        assert(tmp == i);
    }
    return 0;
}
```