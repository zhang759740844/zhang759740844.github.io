title: iOS Crash 收集框架 KSCrash 源码解析(中)
date: 2019/11/1 14:07:12  
categories: iOS
tags: 

 - 学习笔记
 - 源码解析
---

前一篇看了如何捕获各种类型的异常，这一篇将探究如何**解析异常**。这一篇的理解有一定的难度，你需要提前对 Mach-O 的结构有一个大体上的了解。其中涉及到如何将 crash 的地址 symbolicate 的过程，步骤类似于 fishhook，可以翻阅我之前对 fishhook 的解析。

<!--more-->

## Crash 回调

在接收到各种类型的 Crash 信息之后，会统一回调一个处理函数，它就是 `KSCrashC.c` 中的 `onCrash()` 方法：

```c
/// 接收到 crash 信息的处理函数
static void onCrash(struct KSCrash_MonitorContext* monitorContext)
{
    if (monitorContext->currentSnapshotUserReported == false) {
        KSLOG_DEBUG("Updating application state to note crash.");
        kscrashstate_notifyAppCrash();
    }
    monitorContext->consoleLogPath = g_shouldAddConsoleLogToReport ? g_consoleLogPath : NULL;

    if(monitorContext->crashedDuringCrashHandling)
    {
        kscrashreport_writeRecrashReport(monitorContext, g_lastCrashReportFilePath);
    }
    else
    {
        char crashReportFilePath[KSFU_MAX_PATH_LENGTH];
        int64_t reportID = kscrs_getNextCrashReport(crashReportFilePath);
        strncpy(g_lastCrashReportFilePath, crashReportFilePath, sizeof(g_lastCrashReportFilePath));
        /// 将 context 转为 report 写入路径s中
        kscrashreport_writeStandardReport(monitorContext, crashReportFilePath);

        if(g_reportWrittenCallback)
        {
            g_reportWrittenCallback(reportID);
        }
    }
}
```

其中重要的就是讲 context 写入文件的 `kscrashreport_writeStandardReport()` 方法：

```c
/// 将 context 转为 report 写入 path
void kscrashreport_writeStandardReport(const KSCrash_MonitorContext* const monitorContext, const char* const path)
{
    KSLOG_INFO("Writing crash report to %s", path);
    char writeBuffer[1024];
    KSBufferedWriter bufferedWriter;

    if(!ksfu_openBufferedWriter(&bufferedWriter, path, writeBuffer, sizeof(writeBuffer)))
    {
        return;
    }

    ksccd_freeze();
    
    KSJSONEncodeContext jsonContext;
    /// jsonContext 的 userData 是 writer  writer 的 context 是 jsonContext
    jsonContext.userData = &bufferedWriter;
    /// 写入操作的执行者
    KSCrashReportWriter concreteWriter;
    KSCrashReportWriter* writer = &concreteWriter;
    /// 初始化 writer 的各个方法
    prepareReportWriter(writer, &jsonContext);

    ksjson_beginEncode(getJsonContext(writer), true, addJSONData, &bufferedWriter);

    /// 将 “{” 写入 buffer
    writer->beginObject(writer, KSCrashField_Report);
    {
        /// 写入基本信息
        writeReportInfo(writer,
                        KSCrashField_Report,
                        KSCrashReportType_Standard,
                        monitorContext->eventID,
                        monitorContext->System.processName);
        /// 将 buffer 保存到磁盘中
        ksfu_flushBufferedWriter(&bufferedWriter);

        /// 把所有 image 信息写入磁盘
        writeBinaryImages(writer, KSCrashField_BinaryImages);
        ksfu_flushBufferedWriter(&bufferedWriter);

        /// 写入 NSException 出错的信息
        writeProcessState(writer, KSCrashField_ProcessState, monitorContext);
        ksfu_flushBufferedWriter(&bufferedWriter);

        /// 写入系统信息
        writeSystemInfo(writer, KSCrashField_System, monitorContext);
        ksfu_flushBufferedWriter(&bufferedWriter);

        writer->beginObject(writer, KSCrashField_Crash);
        {
            writeError(writer, KSCrashField_Error, monitorContext);
            ksfu_flushBufferedWriter(&bufferedWriter);
            /// 写入所有的线程信息
            writeAllThreads(writer,
                            KSCrashField_Threads,
                            monitorContext,
                            g_introspectionRules.enabled);
            ksfu_flushBufferedWriter(&bufferedWriter);
        }
        writer->endContainer(writer);

        if(g_userInfoJSON != NULL)
        {
            addJSONElement(writer, KSCrashField_User, g_userInfoJSON, false);
            ksfu_flushBufferedWriter(&bufferedWriter);
        }
        else
        {
            writer->beginObject(writer, KSCrashField_User);
        }
        if(g_userSectionWriteCallback != NULL)
        {
            ksfu_flushBufferedWriter(&bufferedWriter);
            if (monitorContext->currentSnapshotUserReported == false) {
                g_userSectionWriteCallback(writer);
            }
        }
        writer->endContainer(writer);
        ksfu_flushBufferedWriter(&bufferedWriter);

        writeDebugInfo(writer, KSCrashField_Debug, monitorContext);
    }
    writer->endContainer(writer);
    
    ksjson_endEncode(getJsonContext(writer));
    ksfu_closeBufferedWriter(&bufferedWriter);
    ksccd_unfreeze();
}
```

### 写入方法

先来看怎么写入的。不用管那些初始化方法，直接看 `beginObject()`，它在兜兜转转后调用了如下方法：

```c
/// 将 data 保存到 buffer 或者文件系统中
static int addJSONData(const char* restrict const data, const int length, void* restrict userData)
{
    KSBufferedWriter* writer = (KSBufferedWriter*)userData;
    const bool success = ksfu_writeBufferedWriter(writer, data, length);
    return success ? KSJSON_OK : KSJSON_ERROR_CANNOT_ADD_DATA;
}

/// 将 data 写入 buffer 或者文件系统中
bool ksfu_writeBufferedWriter(KSBufferedWriter* writer, const char* restrict const data, const int length)
{
    /// 如果 buffer 中的字符数量超出最大值了，那么 flush buffer 到文件系统中
    if(length > writer->bufferLength - writer->position)
    {
        ksfu_flushBufferedWriter(writer);
    }
    if(length > writer->bufferLength)
    {
        return ksfu_writeBytesToFD(writer->fd, data, length);
    }
    /// 否则写入 buffer 中
    memcpy(writer->buffer + writer->position, data, length);
    writer->position += length;
    return true;
}

/// 判断是否要将 buffer 写入文件
bool ksfu_flushBufferedWriter(KSBufferedWriter* writer)
{
    if(writer->fd > 0 && writer->position > 0)
    {
        if(!ksfu_writeBytesToFD(writer->fd, writer->buffer, writer->position))
        {
            return false;
        }
        writer->position = 0;
    }
    return true;
}

bool ksfu_writeBytesToFD(const int fd, const char* const bytes, int length)
{
    const char* pos = bytes;
    while(length > 0)
    {
        int bytesWritten = (int)write(fd, pos, (unsigned)length);
        if(bytesWritten == -1)
        {
            KSLOG_ERROR("Could not write to fd %d: %s", fd, strerror(errno));
            return false;
        }
        length -= bytesWritten;
        pos += bytesWritten;
    }
    return true;
}
```

`KSBufferedWriter` 打开了一个写入流，同时维护了一个 1024 大小的 buffer。当 buffer 中的数据超出之后，就会将 buffer 写入文件中。

### 写入 image 信息

写入 image 信息通过 `writeBinaryImages()` 实现：

```c
static void writeBinaryImages(const KSCrashReportWriter* const writer, const char* const key)
{
    /// 通过 _dyld_image_count 获取 image 的数量
    const int imageCount = ksdl_imageCount();

    writer->beginArray(writer, key);
    {
        for(int iImg = 0; iImg < imageCount; iImg++)
        {
            /// 通过 _dyld_get_image_header 获取 image 信息
            writeBinaryImage(writer, NULL, iImg);
        }
    }
    writer->endContainer(writer);
}

int ksdl_imageCount()
{
    return (int)_dyld_image_count();
}
```

通过 dyld 提供的 `_dyld_image_count()` 方法获取加载的 image 数量。具体的 image 中的信息在 `writeBinaryImage()` 中获取：

```c
static void writeBinaryImage(const KSCrashReportWriter* const writer,
                             const char* const key,
                             const int index)
{
    KSBinaryImage image = {0};
    /// 通过 _dyld_get_image_header 获取 image 信息
    if(!ksdl_getBinaryImage(index, &image))
    {
        return;
    }

    writer->beginObject(writer, key);
    {
        writer->addUIntegerElement(writer, KSCrashField_ImageAddress, image.address);
        writer->addUIntegerElement(writer, KSCrashField_ImageVmAddress, image.vmAddress);
        writer->addUIntegerElement(writer, KSCrashField_ImageSize, image.size);
        writer->addStringElement(writer, KSCrashField_Name, image.name);
        writer->addUUIDElement(writer, KSCrashField_UUID, image.uuid);
        writer->addIntegerElement(writer, KSCrashField_CPUType, image.cpuType);
        writer->addIntegerElement(writer, KSCrashField_CPUSubType, image.cpuSubType);
        writer->addUIntegerElement(writer, KSCrashField_ImageMajorVersion, image.majorVersion);
        writer->addUIntegerElement(writer, KSCrashField_ImageMinorVersion, image.minorVersion);
        writer->addUIntegerElement(writer, KSCrashField_ImageRevisionVersion, image.revisionVersion);
    }
    writer->endContainer(writer);
}

bool ksdl_getBinaryImage(int index, KSBinaryImage* buffer)
{
    /// 通过 _dyld_get_image_header 获取 image 的 header
    const struct mach_header* header = _dyld_get_image_header((unsigned)index);
    if(header == NULL)
    {
        return false;
    }
    
    uintptr_t cmdPtr = firstCmdAfterHeader(header);
    if(cmdPtr == 0)
    {
        return false;
    }
    
    // Look for the TEXT segment to get the image size.
    // Also look for a UUID command.
    uint64_t imageSize = 0;
    uint64_t imageVmAddr = 0;
    uint64_t version = 0;
    uint8_t* uuid = NULL;
    
    for(uint32_t iCmd = 0; iCmd < header->ncmds; iCmd++)
    {
        struct load_command* loadCmd = (struct load_command*)cmdPtr;
        switch(loadCmd->cmd)
        {
            case LC_SEGMENT:
            {
                struct segment_command* segCmd = (struct segment_command*)cmdPtr;
                if(strcmp(segCmd->segname, SEG_TEXT) == 0)
                {
                    imageSize = segCmd->vmsize;
                    imageVmAddr = segCmd->vmaddr;
                }
                break;
            }
            case LC_SEGMENT_64:
            {
                struct segment_command_64* segCmd = (struct segment_command_64*)cmdPtr;
                if(strcmp(segCmd->segname, SEG_TEXT) == 0)
                {
                    imageSize = segCmd->vmsize;
                    imageVmAddr = segCmd->vmaddr;
                }
                break;
            }
            case LC_UUID:
            {
                struct uuid_command* uuidCmd = (struct uuid_command*)cmdPtr;
                uuid = uuidCmd->uuid;
                break;
            }
            case LC_ID_DYLIB:
            {
                
                struct dylib_command* dc = (struct dylib_command*)cmdPtr;
                version = dc->dylib.current_version;
                break;
            }
        }
        cmdPtr += loadCmd->cmdsize;
    }

    buffer->address = (uintptr_t)header;
    buffer->vmAddress = imageVmAddr;
    buffer->size = imageSize;
    buffer->name = _dyld_get_image_name((unsigned)index);
    buffer->uuid = uuid;
    buffer->cpuType = header->cputype;
    buffer->cpuSubType = header->cpusubtype;
    buffer->majorVersion = version >> 16;
    buffer->minorVersion = (version >> 8) & 0xff;
    buffer->revisionVersion = version & 0xff;
    
    return true;
}
```

总的来说，通过 `_dyld_get_image_header` 获取 image 的 header 部分。然后通过 header 位置定位到 load command，遍历 load command 的信息

### 写入线程信息

前一篇讲到过，在 crash 的时候，会 suspend 所有的线程，并且获取它们的基本信息。现在就要对获取到的线程进行解析：

```c
static void writeThread(const KSCrashReportWriter* const writer,
                        const char* const key,
                        const KSCrash_MonitorContext* const crash,
                        const struct KSMachineContext* const machineContext,
                        const int threadIndex,
                        const bool shouldWriteNotableAddresses)
{
    bool isCrashedThread = ksmc_isCrashedContext(machineContext);
    KSThread thread = ksmc_getThreadFromContext(machineContext);
    KSLOG_DEBUG("Writing thread %x (index %d). is crashed: %d", thread, threadIndex, isCrashedThread);

    KSStackCursor stackCursor;
    bool hasBacktrace = getStackCursor(crash, machineContext, &stackCursor);

    writer->beginObject(writer, key);
    {
        if(hasBacktrace)
        {
            /// 写入调用栈
            writeBacktrace(writer, KSCrashField_Backtrace, &stackCursor);
        }
        if(ksmc_canHaveCPUState(machineContext))
        {
            /// 写入寄存器
            writeRegisters(writer, KSCrashField_Registers, machineContext);
        }
        /// 写入线程index
        writer->addIntegerElement(writer, KSCrashField_Index, threadIndex);
        const char* name = ksccd_getThreadName(thread);
        if(name != NULL)
        {
            /// 线程有名称就写入线程名称
            writer->addStringElement(writer, KSCrashField_Name, name);
        }
        name = ksccd_getQueueName(thread);
        if(name != NULL)
        {
            /// 写入 dispatch_queue 的名称
            writer->addStringElement(writer, KSCrashField_DispatchQueue, name);
        }
        /// 是否是崩溃线程
        writer->addBooleanElement(writer, KSCrashField_Crashed, isCrashedThread);
        writer->addBooleanElement(writer, KSCrashField_CurrentThread, thread == ksthread_self());
        /// 如果是崩溃的线程
        if(isCrashedThread)
        {
            /// 将 stack pointer 上的部分数据拷贝出来
            writeStackContents(writer, KSCrashField_Stack, machineContext, stackCursor.state.hasGivenUp);
            if(shouldWriteNotableAddresses)
            {
                /// 将通过 zombie 记录的地址拷贝出来
                writeNotableAddresses(writer, KSCrashField_NotableAddresses, machineContext);
            }
        }
    }
    writer->endContainer(writer);
}
```

其中比较重要的是符号化调用堆栈，写入寄存器的值，以及写入 zombie 记录的信息。

#### 符号化调用堆栈

符号化的过程其实在  fishhock 中有介绍。就是通过实际的地址找到符号表中相应的符号，再到字符串表中找到对应的字符串：

```c
static void writeBacktrace(const KSCrashReportWriter* const writer,
                           const char* const key,
                           KSStackCursor* stackCursor)
{
    writer->beginObject(writer, key);
    {
        writer->beginArray(writer, KSCrashField_Contents);
        {
            /// 对每一层调用栈进行符号化
            while(stackCursor->advanceCursor(stackCursor))
            {
                writer->beginObject(writer, NULL);
                {
                    /// 对调用栈符号化
                    if(stackCursor->symbolicate(stackCursor))
                    {
                        /// 把符号化后的 image 名，地址，symbol 名，地址写入
                        if(stackCursor->stackEntry.imageName != NULL)
                        {
                            writer->addStringElement(writer, KSCrashField_ObjectName, ksfu_lastPathEntry(stackCursor->stackEntry.imageName));
                        }
                        writer->addUIntegerElement(writer, KSCrashField_ObjectAddr, stackCursor->stackEntry.imageAddress);
                        if(stackCursor->stackEntry.symbolName != NULL)
                        {
                            writer->addStringElement(writer, KSCrashField_SymbolName, stackCursor->stackEntry.symbolName);
                        }
                        writer->addUIntegerElement(writer, KSCrashField_SymbolAddr, stackCursor->stackEntry.symbolAddress);
                    }
                    writer->addUIntegerElement(writer, KSCrashField_InstructionAddr, stackCursor->stackEntry.address);
                }
                writer->endContainer(writer);
            }
        }
        writer->endContainer(writer);
        writer->addIntegerElement(writer, KSCrashField_Skipped, 0);
    }
    writer->endContainer(writer);
}
```

其核心逻辑在以下方法中：

```c
/// 根据 address 获取符号名
bool ksdl_dladdr(const uintptr_t address, Dl_info* const info)
{
    info->dli_fname = NULL;
    info->dli_fbase = NULL;
    info->dli_sname = NULL;
    info->dli_saddr = NULL;

    /// 判断 address 是在第几个 image 内
    const uint32_t idx = imageIndexContainingAddress(address);
    if(idx == UINT_MAX)
    {
        return false;
    }
    /// 获得该 image 的 header
    const struct mach_header* header = _dyld_get_image_header(idx);
    /// 获得该 image 的基地址
    const uintptr_t imageVMAddrSlide = (uintptr_t)_dyld_get_image_vmaddr_slide(idx);
    /// 获取 address 相对于 image 的偏移
    const uintptr_t addressWithSlide = address - imageVMAddrSlide;
    /// 获得 segment 在虚拟内存中的基地址
    const uintptr_t segmentBase = segmentBaseOfImageIndex(idx) + imageVMAddrSlide;
    if(segmentBase == 0)
    {
        return false;
    }

    /// 获取 image 的名字
    info->dli_fname = _dyld_get_image_name(idx);
    /// 获取 header 地址
    info->dli_fbase = (void*)header;

    // Find symbol tables and get whichever symbol is closest to the address.
    const STRUCT_NLIST* bestMatch = NULL;
    uintptr_t bestDistance = ULONG_MAX;
    uintptr_t cmdPtr = firstCmdAfterHeader(header);
    if(cmdPtr == 0)
    {
        return false;
    }
    for(uint32_t iCmd = 0; iCmd < header->ncmds; iCmd++)
    {
        const struct load_command* loadCmd = (struct load_command*)cmdPtr;
        if(loadCmd->cmd == LC_SYMTAB)
        {
            /// 找到 symbol table 的 load command
            const struct symtab_command* symtabCmd = (struct symtab_command*)cmdPtr;
            /// 通过 load command 中的 offset + segment 的基地址，得到 symbol table 的实际地址
            const STRUCT_NLIST* symbolTable = (STRUCT_NLIST*)(segmentBase + symtabCmd->symoff);
            /// 找到 string table 的位置
            const uintptr_t stringTable = segmentBase + symtabCmd->stroff;

            /// 找到最佳的符号
            for(uint32_t iSym = 0; iSym < symtabCmd->nsyms; iSym++)
            {
                // If n_value is 0, the symbol refers to an external object.
                if(symbolTable[iSym].n_value != 0)
                {
                    uintptr_t symbolBase = symbolTable[iSym].n_value;
                    ///
                    uintptr_t currentDistance = addressWithSlide - symbolBase;
                    if((addressWithSlide >= symbolBase) &&
                       (currentDistance <= bestDistance))
                    {
                        bestMatch = symbolTable + iSym;
                        bestDistance = currentDistance;
                    }
                }
            }
            /// 根据符号的位置找到其在 string table 中表示的字符串
            if(bestMatch != NULL)
            {
                info->dli_saddr = (void*)(bestMatch->n_value + imageVMAddrSlide);
                if(bestMatch->n_desc == 16)
                {
                    // This image has been stripped. The name is meaningless, and
                    // almost certainly resolves to "_mh_execute_header"
                    info->dli_sname = NULL;
                }
                else
                {
                    info->dli_sname = (char*)((intptr_t)stringTable + (intptr_t)bestMatch->n_un.n_strx);
                    if(*info->dli_sname == '_')
                    {
                        info->dli_sname++;
                    }
                }
                break;
            }
        }
        cmdPtr += loadCmd->cmdsize;
    }
    
    return true;
}
```

代码中已经做了详尽的注释，对于原理我就不再做详尽的阐述了。如果对这部分不明白，也可以参考我对于 fishhook 的解析。总之，在这个过程中就完成了从地址到 image 中符号的转化。

#### 写入寄存器

在通过 crash 获取的 context 信息中，我们可以拿到寄存器相关的信息：

```c
static void writeBasicRegisters(const KSCrashReportWriter* const writer,
                                const char* const key,
                                const struct KSMachineContext* const machineContext)
{
    char registerNameBuff[30];
    const char* registerName;
    writer->beginObject(writer, key);
    {
        const int numRegisters = kscpu_numRegisters();
        for(int reg = 0; reg < numRegisters; reg++)
        {
            registerName = kscpu_registerName(reg);
            if(registerName == NULL)
            {
                snprintf(registerNameBuff, sizeof(registerNameBuff), "r%d", reg);
                registerName = registerNameBuff;
            }
            writer->addUIntegerElement(writer, registerName,
                                       kscpu_registerValue(machineContext, reg));
        }
    }
    writer->endContainer(writer);
}
```

获取到的寄存器有以下所示：

```c
static const char* g_registerNames[] =
{
    "rax", "rbx", "rcx", "rdx",
    "rdi", "rsi",
    "rbp", "rsp",
    "r8", "r9", "r10", "r11", "r12", "r13", "r14", "r15",
    "rip", "rflags",
    "cs", "fs", "gs"
};
```

根据寄存器的序号通过如下方法在 context 中获取寄存器中存储的值：

```c
uint64_t kscpu_registerValue(const KSMachineContext* const context, const int regNumber)
{
    switch(regNumber)
    {
        case 0:
            return context->machineContext.__ss.__rax;
        case 1:
            return context->machineContext.__ss.__rbx;
        case 2:
            return context->machineContext.__ss.__rcx;
        case 3:
            return context->machineContext.__ss.__rdx;
        case 4:
            return context->machineContext.__ss.__rdi;
        case 5:
            return context->machineContext.__ss.__rsi;
        case 6:
            return context->machineContext.__ss.__rbp;
        case 7:
            return context->machineContext.__ss.__rsp;
        case 8:
            return context->machineContext.__ss.__r8;
        case 9:
            return context->machineContext.__ss.__r9;
        case 10:
            return context->machineContext.__ss.__r10;
        case 11:
            return context->machineContext.__ss.__r11;
        case 12:
            return context->machineContext.__ss.__r12;
        case 13:
            return context->machineContext.__ss.__r13;
        case 14:
            return context->machineContext.__ss.__r14;
        case 15:
            return context->machineContext.__ss.__r15;
        case 16:
            return context->machineContext.__ss.__rip;
        case 17:
            return context->machineContext.__ss.__rflags;
        case 18:
            return context->machineContext.__ss.__cs;
        case 19:
            return context->machineContext.__ss.__fs;
        case 20:
            return context->machineContext.__ss.__gs;
    }

    KSLOG_ERROR("Invalid register number: %d", regNumber);
    return 0;
}
```

#### 记录调用堆栈的部分信息

分析 crash 的原因，怎么能少了出错线程堆栈的解析呢？KSCrash 提供了方法将出错线程的堆栈上的部分信息拷贝出来：

```c
static void writeStackContents(const KSCrashReportWriter* const writer,
                               const char* const key,
                               const struct KSMachineContext* const machineContext,
                               const bool isStackOverflow)
{
    /// 拿到 stack pointer
    uintptr_t sp = kscpu_stackPointer(machineContext);
    if((void*)sp == NULL)
    {
        return;
    }

    uintptr_t lowAddress = sp + (uintptr_t)(kStackContentsPushedDistance * (int)sizeof(sp) * kscpu_stackGrowDirection() * -1);
    uintptr_t highAddress = sp + (uintptr_t)(kStackContentsPoppedDistance * (int)sizeof(sp) * kscpu_stackGrowDirection());
    if(highAddress < lowAddress)
    {
        uintptr_t tmp = lowAddress;
        lowAddress = highAddress;
        highAddress = tmp;
    }
    writer->beginObject(writer, key);
    {
        writer->addStringElement(writer, KSCrashField_GrowDirection, kscpu_stackGrowDirection() > 0 ? "+" : "-");
        writer->addUIntegerElement(writer, KSCrashField_DumpStart, lowAddress);
        writer->addUIntegerElement(writer, KSCrashField_DumpEnd, highAddress);
        writer->addUIntegerElement(writer, KSCrashField_StackPtr, sp);
        writer->addBooleanElement(writer, KSCrashField_Overflow, isStackOverflow);
        uint8_t stackBuffer[kStackContentsTotalDistance * sizeof(sp)];
        int copyLength = (int)(highAddress - lowAddress);
        /// 拷贝 lowAddress 上的数据到 buffer 中
        if(ksmem_copySafely((void*)lowAddress, stackBuffer, copyLength))
        {
            writer->addDataElement(writer, KSCrashField_Contents, (void*)stackBuffer, copyLength);
        }
        else
        {
            writer->addStringElement(writer, KSCrashField_Error, "Stack contents not accessible");
        }
    }
    writer->endContainer(writer);
}
```

拿到 stack pointer 是直接通过 sp 寄存器：

```c
uintptr_t kscpu_stackPointer(const KSMachineContext* const context)
{
    return context->machineContext.__ss.__rsp;
}
```

拷贝栈上的信息的范围是：

```c
#define kStackContentsPushedDistance 20
#define kStackContentsPoppedDistance 10
#define kStackContentsTotalDistance (kStackContentsPushedDistance + kStackContentsPoppedDistance)
```

这个是作者设置的，也就说是，拷贝栈内 20 个对象，以及刚刚出栈的 10 个对象的地址。在拿到 sp 和范围之后，就可以通过 c 的方法获取：

```c
static inline int copySafely(const void* restrict const src, void* restrict const dst, const int byteCount)
{
    vm_size_t bytesCopied = 0;
    kern_return_t result = vm_read_overwrite(mach_task_self(),
                                             (vm_address_t)src,
                                             (vm_size_t)byteCount,
                                             (vm_address_t)dst,
                                             &bytesCopied);
    if(result != KERN_SUCCESS)
    {
        return 0;
    }
    return (int)bytesCopied;
}
```

#### 取出地址上的对象

上面无论是寄存器还是堆栈，取出的都是地址。但是我们其实更需要的是对象的信息。因此，我们还需要到地址上去解析对象信息：

```c
static void writeNotableAddresses(const KSCrashReportWriter* const writer,
                                  const char* const key,
                                  const struct KSMachineContext* const machineContext)
{
    writer->beginObject(writer, key);
    {
        /// 获取 register 上的对象
        writeNotableRegisters(writer, machineContext);
        /// 获取 stack 上的对象
        writeNotableStackContents(writer,
                                  machineContext,
                                  kStackNotableSearchBackDistance,
                                  kStackNotableSearchForwardDistance);
    }
    writer->endContainer(writer);
}

```

我们以解析寄存器上的对象为例：

```c
static void writeMemoryContents(const KSCrashReportWriter* const writer,
                                const char* const key,
                                const uintptr_t address,
                                int* limit)
{
    (*limit)--;
    const void* object = (const void*)address;
    writer->beginObject(writer, key);
    {
        writer->addUIntegerElement(writer, KSCrashField_Address, address);
        writeZombieIfPresent(writer, KSCrashField_LastDeallocObject, address);
        if(!writeObjCObject(writer, address, limit))
        {
            if(object == NULL)
            {
                writer->addStringElement(writer, KSCrashField_Type, KSCrashMemType_NullPointer);
            }
            else if(isValidString(object))
            {
                writer->addStringElement(writer, KSCrashField_Type, KSCrashMemType_String);
                writer->addStringElement(writer, KSCrashField_Value, (const char*)object);
            }
            else
            {
                writer->addStringElement(writer, KSCrashField_Type, KSCrashMemType_Unknown);
            }
        }
    }
    writer->endContainer(writer);
}
```

`writeObjCObject()` 将地址转化为对象：

```c
static bool writeObjCObject(const KSCrashReportWriter* const writer,
                            const uintptr_t address,
                            int* limit)
{
#if KSCRASH_HAS_OBJC
    const void* object = (const void*)address;
    switch(ksobjc_objectType(object))
    {
        case KSObjCTypeClass:
            writer->addStringElement(writer, KSCrashField_Type, KSCrashMemType_Class);
            writer->addStringElement(writer, KSCrashField_Class, ksobjc_className(object));
            return true;
        case KSObjCTypeObject:
        {
            writer->addStringElement(writer, KSCrashField_Type, KSCrashMemType_Object);
            const char* className = ksobjc_objectClassName(object);
            writer->addStringElement(writer, KSCrashField_Class, className);
            if(!isRestrictedClass(className))
            {
                switch(ksobjc_objectClassType(object))
                {
                    case KSObjCClassTypeString:
                        writeNSStringContents(writer, KSCrashField_Value, address, limit);
                        return true;
                    case KSObjCClassTypeURL:
                        writeURLContents(writer, KSCrashField_Value, address, limit);
                        return true;
                    case KSObjCClassTypeDate:
                        writeDateContents(writer, KSCrashField_Value, address, limit);
                        return true;
                    case KSObjCClassTypeArray:
                        if(*limit > 0)
                        {
                            writeArrayContents(writer, KSCrashField_FirstObject, address, limit);
                        }
                        return true;
                    case KSObjCClassTypeNumber:
                        writeNumberContents(writer, KSCrashField_Value, address, limit);
                        return true;
                    case KSObjCClassTypeDictionary:
                    case KSObjCClassTypeException:
                        // TODO: Implement these.
                        if(*limit > 0)
                        {
                            writeUnknownObjectContents(writer, KSCrashField_Ivars, address, limit);
                        }
                        return true;
                    case KSObjCClassTypeUnknown:
                        if(*limit > 0)
                        {
                            writeUnknownObjectContents(writer, KSCrashField_Ivars, address, limit);
                        }
                        return true;
                }
            }
            break;
        }
        case KSObjCTypeBlock:
            writer->addStringElement(writer, KSCrashField_Type, KSCrashMemType_Block);
            const char* className = ksobjc_objectClassName(object);
            writer->addStringElement(writer, KSCrashField_Class, className);
            return true;
        case KSObjCTypeUnknown:
            break;
    }
#endif

    return false;
}
```

这一段代码有点长，先把 address 强转为一个 object，然后判断 object 到底是 tagged pointer 还是 block 还是 OC 类型，还是自建的 Class。根据类型写入相应的信息。

### 野指针的监控

上面的获取地址对象方法中，我们要注意到一个方法 `writeZombieIfPresent()`。这是 KSCrash 提供的 Zombie，用于监控野指针。我们来看一下它的实现：

```c
static void writeZombieIfPresent(const KSCrashReportWriter* const writer,
                                 const char* const key,
                                 const uintptr_t address)
{
#if KSCRASH_HAS_OBJC
    const void* object = (const void*)address;
    const char* zombieClassName = kszombie_className(object);
    if(zombieClassName != NULL)
    {
        writer->addStringElement(writer, key, zombieClassName);
    }
#endif
}

const char* kszombie_className(const void* object)
{
    volatile Zombie* cache = g_zombieCache;
    if(cache == NULL || object == NULL)
    {
        return NULL;
    }

    Zombie* zombie = (Zombie*)cache + hashIndex(object);
    if(zombie->object == object)
    {
        return zombie->className;
    }
    return NULL;
}
```

它的方法非常简洁。就是判断当前 address 上的 object 是否是记录过的对象。这是怎么做到的呢？`g_zombieCache` 又是什么呢？这就需要回到 `KSCrashMonitor_Zombie.c` 中查看。

前一篇我们说到，每一个 monitor 会被调用其 `setEnabled()` 方法启动。Zombie 也不例外。它在 `setEnabled()` 中调用了 `install` 方法：

```c
static void install()
{
    unsigned cacheSize = CACHE_SIZE;
    g_zombieHashMask = cacheSize - 1;
    g_zombieCache = calloc(cacheSize, sizeof(*g_zombieCache));
    if(g_zombieCache == NULL)
    {
        KSLOG_ERROR("Error: Could not allocate %u bytes of memory. KSZombie NOT installed!",
              cacheSize * sizeof(*g_zombieCache));
        return;
    }

    g_lastDeallocedException.class = objc_getClass("NSException");
    g_lastDeallocedException.address = NULL;
    g_lastDeallocedException.name[0] = 0;
    g_lastDeallocedException.reason[0] = 0;

    installDealloc_NSObject();
    installDealloc_NSProxy();
}

CREATE_ZOMBIE_HANDLER_INSTALLER(NSObject)
CREATE_ZOMBIE_HANDLER_INSTALLER(NSProxy)
  
#define CREATE_ZOMBIE_HANDLER_INSTALLER(CLASS) \
static IMP g_originalDealloc_ ## CLASS; \
static void handleDealloc_ ## CLASS(id self, SEL _cmd) \
{ \
    handleDealloc(self); \
    typedef void (*fn)(id,SEL); \
    fn f = (fn)g_originalDealloc_ ## CLASS; \
    f(self, _cmd); \
} \
static void installDealloc_ ## CLASS() \
{ \
    Method method = class_getInstanceMethod(objc_getClass(#CLASS), sel_registerName("dealloc")); \
    g_originalDealloc_ ## CLASS = method_getImplementation(method); \
    method_setImplementation(method, (IMP)handleDealloc_ ## CLASS); \
}
```

根据代码，我们可以知道，它 hook 了 `NSObject` 以及 `NSProxy` 的 dealloc 方法。先调用自己的处理方法 `handleDealloc()` 然后再调用原来的 dealloc 方法。注意这里调用原来 dealloc 方法的实现，这样做的原因我在 JSPatch 中有分析过，这里不做赘述了：

```c
typedef void (*fn)(id,SEL); \
fn f = (fn)g_originalDealloc_ ## CLASS; \
f(self, _cmd); \
```

再来看 `handleDealloc()` 方法：

```c
static inline void handleDealloc(const void* self)
{
    volatile Zombie* cache = g_zombieCache;
    likely_if(cache != NULL)
    {
        /// 把要销毁的对象保存在 cache 中。
        /// hashIndex 就是对要销毁对象求 hash，找到它在 cache 中应该存在的位置
        Zombie* zombie = (Zombie*)cache + hashIndex(self);
        zombie->object = self;
        Class class = object_getClass((id)self);
        zombie->className = class_getName(class);
        for(; class != nil; class = class_getSuperclass(class))
        {
            /// 如果要销毁的是 NSException，那么把它保存起来
            unlikely_if(class == g_lastDeallocedException.class)
            {
                storeException(self);
            }
        }
    }
}
```

在 `install` 的时候的时候创建了一个空的 Cache：

```c
g_zombieCache = calloc(cacheSize, sizeof(*g_zombieCache));
```

在处理 dealloc 的时候，会把对象的地址放到这个 Zombie 的 cache 中。这样的作用就是对于已经销毁的对象，我们记录了一份它们的地址信息，这样以后出现野指针 crash 的时候，如果发现是 zombie 中指向的对象，那么就可以说明它被提前释放了。当然，这并不是非常准确的，因为 hash 获取 index 的方式总是会产生一定的碰撞导致对象被覆盖。当然不可否认这是一种经济有效的测试野指针的方式。作者自己都在注释中说明这是一种 Poor man's Zombie tracking XD

```c
/* Poor man's zombie tracking.
 *
 * Benefits:
 * - Very low CPU overhead.
 * - Low memory overhead.
 *
 * Limitations:
 * - Not guaranteed to catch all zombies.
 * - Can generate false positives or incorrect class names.
 * - KSZombie itself must be compiled with ARC disabled. You can enable ARC in
 *   your app, but KSZombie must be compiled in a separate library if you do.
 */
```

## 总结

到这里，KSCrash 的中篇结束了。这一篇因为涉及的方法和 API 太多，而且很多都是之前研究 fishhook 的时候解释过的，所以我对这部分内容的解释会显得比较少。当然，对于一般的开发者来说，掌握其中的基本原理就已经足够了，对于其中很多 c 相关的 API 以及写法没有必要纠结太深。

KSCrash 分析的部分应该就是这样了。在 KSCrash 下中，我打算去了解一下腾讯开源的 Matrix 中内存预警相关的原理。







