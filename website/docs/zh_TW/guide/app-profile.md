# App Profile

App Profile 是 KernelSU 提供的一種機制，用於自訂各種應用程式的配置。

對於被授予 root 權限的應用程式（即能夠使用`su`），App Profile 也可以稱為 Root Profile。它允許自訂 `su` 命令的 `uid`、`gid`、`groups`、`capability` 和 `SELinux context`，從而限制 root 使用者的權限。例如，它可以僅向防火牆應用程式授予網路權限，同時拒絕檔案存取權限，或者可以為凍結應用程式授予 shell 權限而不是完全 root 存取權限：**以最小特權原則限制權力。**

對於沒有 `root` 權限的普通應用程式，App Profile 可以控制內核和模組系統對這些應用程式的行為。例如，它可以確定是否應解決由模組引起的修改。核心和模組系統可以根據此配置做出決策，例如執行類似於`隱藏`的操作

## Root Profile

### UID、GID 和群組

Linux系統有使用者和群組兩個概念。每個使用者都有一個使用者 ID (UID)，一個使用者可以屬於多個群組，每個群組都有自己的群組 ID (GID)。這些 ID 用於識別系統中的使用者並確定他們可以存取哪些系統資源。

UID 為 0 的使用者稱為 root 使用者，GID 為 0 的群組稱為 root 群組。 root 使用者群組通常擁有最高的系統權限。

對於 Android 系統來說，每個應用程式都是一個獨立的使用者（不考慮 share UID 場景），擁有唯一的 UID。比如 `0` 代表 root 使用者，`1000` 代表 `system`，`2000` 代表 ADB shell，10000 到 19999 代表普通使用者。

::: INFO
這裡提到的 UID 與 Android 系統中的多使用者或工作設定檔（Work Profile）的概念並不相同。工作設定檔實際上是透過劃分UID範圍來實現的。例如，10000-19999 代表主使用者，而 110000-119999 代表工作設定檔。其中每個普通應用程式都有自己唯一的UID。
:::

每個應用程式可以有多個群組，GID 代表主要群組，通常與 UID 相符。其他組別稱為補充組。某些權限透過群組進行控制，例如網路存取權限或藍牙存取。

例如，如果我們在 ADB shell 中執行 `id` 命令，輸出可能如下所示：

```sh
oriole:/ $ id
uid=2000(shell) gid=2000(shell) groups=2000(shell),1004(input),1007(log),1011(adb),1015(sdcard_rw),1028(sdcard_r),1078(ext_data_rw),1079(ext_obb_rw),3001(net_bt_admin),3002(net_bt),3003(inet),3006(net_bw_stats),3009(readproc),3011(uhid),3012(readtracefs) context=u:r:shell:s0
```

這裡，UID是 `2000`，GID（主群組 ID）也是 `2000`。此外，它還屬於幾個補充組，例如 `inet`（指示創建 `AF_INET` 和 `AF_INET6` 套接字的能力）和 `sdcard_rw`（指示SD卡的讀取/寫入權限）。

KernelSU 的 Root Profile 允許在執行 `su` 後自訂根程式的 UID、GID 和群組。例如，根應用程式的 Root Profile 可以將其 UID 設定為`2000`，這表示當使用 `su` 時，該應用程式的實際權限位於 ADB shell 層級。可以刪除 `inet` 群組，以防止 `su` 命令存取網路。

:::tip Note
App Profile僅只限制使用 `su` 後的權限；它不控制應用程式本身的權限。如果應用程式請求了網路存取權限，即使不使用 `su` ，它仍然可以存取網路。從 `su` 中刪除 `inet` 群組只會阻止 `su` 存取網路。
:::

Root Profile 在核心中強制執行，不依賴 root 應用程式的自願行為，與透過 `su` 切換使用者或群組不同， `su` 權限的授予完全取決於使用者而不是開發人員。

### 逃逸

如果 Root 的權限設定不正確，則可能會發生逃逸： Root Profile 的限制會意外失效。

例如，如果您向 ADB shell 使用者授予 root 權限（這是常見情況），然後向常規應用程式授予 root 權限，但將其 root Profile 中的 UID 為 2000（這是 ADB shell 使用者的 UID） ，應用程式可以透過執行兩次 `su` 命令來獲得完整的 root 權限：

1. 第一次`su`，執行受App Profile的強制執行，並將切換到UID `2000（adb shell）` 而不是 `0（root）`。
2. 第二次 `su` 執行，由於 UID 為 `2000`，並且您在App Profile中已授予 UID `2000（adb shell）`root 存取權限，因此應用程式將獲得完全 root 權限！

:::warning 注意
此行為完全是預期的，而不是 BUG！因此，我們建議如下：

如果您確實需要向 ADB 授予 root 權限（例如，作為開發人員），則不建議在配置 Root Profile 時將 UID 變更為 `2000`。使用 `1000（system）` 將是更好的選擇。:::

### 權限

Capabilities 是 Linux 中的一種分權機制。

為了執行權限檢查，傳統的 UNIX 實作區分兩類程式：特權程式（其有效使用者 ID 為 0，稱為超級使用者或 root）和非特權程式（其有效 UID 不為零）。特權程式繞過所有核心權限檢查，而非特權程式則受到基於進程憑證（通常：有效 UID、有效 GID 和補充群組清單）的完全權限檢查。

從 Linux 2.2 開始，Linux 將傳統上與超級使用者相關的權限劃分為不同的單元，稱為 Capabilities，可以獨立啟用和停用。

每項能力代表一個或多個特權。例如，`CAP_DAC_READ_SEARCH` 表示能夠繞過檔案讀取權限檢查，以及目錄讀取和執行權限。如果有效 UID 為 `0` 的使用者（root 使用者）缺乏 `CAP_DAC_READ_SEARCH` 或更高的能力，這意味著即使他們是 root，也無法隨意讀取檔案。

KernelSU 的 Root Profile 允許在執行 `su` 後自訂根程式的 Capability，從而實現部分授予 `root權限` 。與前面提到的 UID 和 GID 不同，某些根應用程式在使用 `su` 後需要 UID 為 `0`。在這種情況下，使用 UID `0` 限制此 root 使用者的能力可以限制其允許的操作。

:::tip 強烈推薦
Linux的Capability[官方文件](https://man7.org/linux/man-pages/man7/capability.7.html)提供了每個Capability所代表的能力的詳細解釋。如果您打算自訂功能，強烈建議您先閱讀本文檔。
:::

## Non Root Profile
### 卸载模組
KernelSU 提供了一種 systemless 的方式來修改系統分區，這是透過掛載 overlayfs 來實現的。但有些情況下，App 可能會對這種行為比較敏感；因此，我們可以透過設定「卸載模組」來卸載掛載在這些 App 上的模組。

另外，KernelSU 管理器的設定介面還提供了一個「預設卸載模組」的開關，這個開關預設是開啟的，這意味著如果不對 App 做額外的設置，預設情況下 KernelSU 或者某些模組會對此 App 執行卸載操作。當然，如果你不喜歡這個設定或這個設定會影響某些 App，你可以有以下選擇：

1. 保持「預設卸載模組」的開關，然後針對不需要「卸載模組」的 App 進行單獨的設置，在 App Profile 中關閉「卸載模組」；（相當於「白名單」）。
2. 關閉「預設卸載模組」的開關，然後針對需要「卸載模組」的 App 進行單獨的設置，在 App Profile 中開啟「卸載模組」；（相當於「黑名單」）。

::: INFO
KernelSU 在 5.10 及以上內核上，內核會執行“卸載模組”的操作；但在 5.10 以下的設備上，這個開關僅僅是一個“設定”，KernelSU 本身不會做任何動作，一些模組（如 Zygisksu 會透過這個模組決定是否需要卸載）:::
