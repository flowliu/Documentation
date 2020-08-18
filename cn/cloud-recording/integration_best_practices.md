
---
title: 云端录制集成最佳实践
description: 
platform: All Platforms
updatedAt: Tue Aug 18 2020 06:09:57 GMT+0800 (CST)
---
# 云端录制集成最佳实践
为了保障录制服务的可靠性，Agora 建议你在集成云端录制 RESTful API 时注意以下几点：

## 使用双域名

<div class="alert note">如果要使用域名 <code>api.agora.cn</code> 发起 RESTful API 请求，必须保证发起请求的服务器位于中国大陆。非中国大陆地区仅支持域名 <code>api.agora.io</code>。</div>

如果你使用域名 `api.agora.io` 发起 RESTful API 请求失败，可以先用该域名重试一次；如再次失败，可将域名替换为 `api.agora.cn`，再次发送请求。建议使用退避策略，如第一次等待 1 秒后重试，第二次等待 3 秒后重试、第三次等待 6 秒后重试，以免超过 QPS 限制导致失败。

## `start` 请求失败后的处理方式

如果 `start` 请求失败，需要根据状态码采取相应措施：

- 如果返回的 HTTP 状态码为 `40x`，则表示请求参数错误，需要进行排查。

- 如果返回的 HTTP 状态码为 `50x`，可使用相同的参数重试多次，直到成功返回 `sid` 为止。建议使用退避策略，如第一次等待 3 秒后重试，第二次等待 6 秒后重试、第三次等待 9 秒后重试，以免超过 QPS 限制导致失败。

  <div class="alert note">使用 <code>acquire</code> 方法获得的 resource ID 时效为 5 分钟；5 分钟后，如果 <code>start</code> 请求仍然返回 <code>50x</code> 错误，建议更换 UID 再次调用 <code>acquire</code>， 获得一个新的 resource ID，并用该 resource ID 再次调用 <code>start</code></code> 方法。</div>

## <a name="query_fail"></a>`query` 请求失败后的处理方式

获得 `sid` 后，即表示 `start` 请求成功，但不代表录制服务已经启动。如需确认录制已成功启动，可以调用 `query` 方法。如果 `query` 请求失败，需要根据状态码采取相应措施：

- 如果返回的 HTTP 状态码一直为 `40x`，则表示请求参数错误，需要进行排查。

- 如果返回的 HTTP 状态码为 `404`，且已经确认请求参数无误，则表示录制并未成功启动、或启动后中途退出。建议采用退避策略多次调用 `query` （例如间隔 5 秒、10 秒、15 秒）进行确认。确认后，你可以根据自己的业务逻辑选择是否要重新启动录制。

  <div class="alert note">此种方法有小于万分之一的误判率，即将一个正在进行的录制进程判定为已结束。因此，建议你在重新启动录制时使用一个新的 UID，以避免频道内两个相同 UID 互踢的情况。</div>

- 如果返回的 HTTP 状态码为 `50x`，则表示 `query` 请求失败，但尚不明确录制是否已退出。建议继续使用退避策略 (5 秒, 10 秒, 15 秒，30 秒) 多次调用 `query`。如果最终返回 `20x` 则表示请求成功，返回 `40x` 需要继续排查。

## 使用 `query` 请求获取 M3U8 文件名

你可以通过 `query` 请求获得 M3U8 文件的文件名。M3U8 文件名是第一份切片文件生成后产生的，所以需要在第一次切片完成后查询。详见[切片规则](https://docs.agora.io/cn/cloud-recording/cloud_recording_manage_files?platform=All%20Platforms#切片规则)。

合流录制模式下，建议在成功开启云端录制 15 秒后调用 `query` 查询 M3U8 文件名；单流录制模式下，建议在成功开启云端录制 1 分钟后调用 query 查询 M3U8 文件名。如果第一次查询失败，可以在 1 分钟后重试。

合流录制模式下，M3U8 文件名的格式为 `<sid>_<cname>.m3u8`。因此，你还可以通过拼接的方式获取切片文件名。详见[录制文件命名规则](https://docs.agora.io/cn/cloud-recording/cloud_recording_manage_files?platform=All%20Platforms#合流模式)。

## 避免录制服务频繁退出

`start` 方法中的 `maxIdleTime` 参数默认值为 30 秒。如果实际场景中主播频繁上下线，过短的 `maxIdleTime` 会导致录制服务频繁退出。对于要求录制服务一直在频道中的场景，需要加大 `maxIdleTime`，避免因频道内短时间无发流端导致的录制退出。

举例来说，如果每堂课中固定有 5 分钟的休息时间，则可以将 `maxIdleTime` 设置为 10 分钟，保证整堂课的录制不中断。

## 保证录制服务状态正常

你可以通过周期性调用 `query` 或冗余消息通知来确认录制服务正在进行中且状态正常。消息通知服务和 `query` 各有优劣，详见[消息通知服务和 query 方法的对比](https://confluence.agoralab.co/pages/viewpage.action?pageId=681271989)。

### 周期性频道查询

如果你对状态查询的可靠性要求较高，Agora 强烈建议你使用 `query` 方法周期性查询频道内的录制状态，例如每隔 1 分钟调用一次 `query`，并根据返回的 HTTP 状态码采取相应措施。详见[如何处理 query 请求失败](#query_fail)。

### 冗余消息通知服务

如果你依赖消息通知服务来监测录制服务状态，建议联系 [support@agora.io](mailto:support@agora.io) 开通冗余消息功能，即接收双路消息通知，降低消息丢失的概率。开通冗余消息功能后，需要你基于 `sid` 对消息进行去重。举例来说，如果你需要在录制超时退出后再次开启录制，流程为：

1. 你的服务器收到 `31`、`32`、或 `11` 事件，表示录制服务已正常退出。
2. 收到事件后，你的应用调用 `acquire`，重新开启录制服务。
3. 在此期间你的服务器又收到了 `31`、`32`、或 `11` 事件。如果以上事件中的 `sid` 与前一次的 `31`、`32`、或 `11` 事件一致，则可以作为冗余事件通知忽略。
4. 如果你需要完全确保成功开启了录制服务，则仍然需要调用 `query` 进行查询。如果 `query` 失败，需要根据返回的 HTTP 状态码采取相应措施。详见[如何处理 query 请求失败](#query_fail)。

