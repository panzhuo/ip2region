# ip2region 运行与测试留痕

- 项目：https://github.com/lionsoul2014/ip2region
- 提交：`6a276cc7c4f9f2d7ff1d8ace14b91af59752f943`（`add missing isp`），共 1402 次提交，标签至 `v3.9.0`
- Java binding 版本：`3.3.7`
- 本地路径：`/Users/panzhuo/Documents/openclaw/ip2region`（完整克隆）
- 测试时间：2026-10-28 11:16 首次；2026-10-29 13:47 在完整克隆上复跑
- 结论：**可用**。构建成功，官方 29 个单元测试全绿，CLI 查询正确，1,033,720 条源数据全量比对零误差。

---

## 1. 环境

| 项 | 版本 |
| --- | --- |
| macOS | Apple Silicon (arm64) |
| JDK（编译/运行） | Zulu 1.8.0_452 |
| Maven | 3.9.9（依赖走阿里云镜像） |
| 数据 | `data/ip2region_v4.xdb`(11 MB)、`data/ip2region_v6.xdb`(36 MB)、`data/ipv4_source.txt`(35 MB) |

## 2. 代码获取

10-28 首次获取时 GitHub 直连仅约 6 KB/s 且频繁断流，采用 `ghfast.top` 加速域做 blob-less 部分克隆 + 稀疏检出 + npm 镜像取数据，并用 git blob hash 校验数据文件逐字节等于上游：

| 文件 | 仓库 blob hash | 实测 hash | 结果 |
| --- | --- | --- | --- |
| `data/ip2region_v4.xdb` | `bda1438a83e9ab152f5451d93e2138e35a166a26` | 同左 | ✅ |
| `data/ip2region_v6.xdb` | `15002d535f31336cc556b672a617186c827ea14c` | 同左 | ✅ |

10-29 网络恢复（直连 4.7 MB/s）后改为完整克隆：

```
git clone https://github.com/lionsoul2014/ip2region.git
git fetch --unshallow
```

## 3. 构建

```
cd binding/java
mvn -B clean package
```

- 完整克隆环境：`BUILD SUCCESS`，6.251 s
- 产物：`target/ip2region-3.3.7.jar`（可执行，Main-Class = `org.lionsoul.ip2region.SearcherTest`）、`-sources.jar`、`-javadoc.jar`

日志：[01-maven-build.log](./01-maven-build.log)

## 4. 单元测试

```
mvn -B test
```

```
Tests run: 29, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

覆盖类：`UtilTest`(3)、`LittleEndianTest`(1)、`IPv4Test`(1)、`BufferTest`(6)、`VersionTest`(2)、`SearcherPoolTest`(3)、`ConfigTest`(8)、`Ip2RegionTest`(5)。

日志：[02-maven-test.log](./02-maven-test.log)

## 5. CLI 查询测试

```
java -jar target/ip2region-3.3.7.jar search
```

| 输入 | 返回 region | 耗时 |
| --- | --- | --- |
| `113.92.157.29` | `中国\|广东省\|深圳市\|电信\|CN` | 10117 μs（首次，含初始化） |
| `223.5.5.5` | `中国\|浙江省\|杭州市\|阿里\|CN` | 125 μs |
| `8.8.8.8` | `United States\|California\|0\|Google LLC\|US` | 230 μs |
| `1.1.1.1` | `Australia\|Queensland\|Brisbane\|0\|AU` | 27 μs |
| `127.0.0.1` | `Reserved\|Reserved\|Reserved\|0\|0` | 34 μs |
| `192.168.1.1` | `Reserved\|Reserved\|Reserved\|0\|0` | 23 μs |
| `240e:3b7:3272:d8d0:db09:c067:8d59:539e` | `中国\|广东省\|深圳市\|电信\|CN` | 507 μs |
| `2001:4860:4860::8888` | `United States\|Florida\|Miami\|Google LLC\|US` | 78 μs |
| `999.999.999.999` | `Reserved\|Reserved\|Reserved\|0\|0` | 12566 μs |

观察项：非法 IP 字符串不抛异常，落到 `Reserved` 段（IPv6 解析路径的容错行为）；业务侧需自行做格式校验。

日志：[03-cli-search.log](./03-cli-search.log)

## 6. 基准校验（全量正确性 + 性能）

`bench` 会用 `data/ipv4_source.txt` 中每条区间的起止 IP 各查一次，region 必须与源文件逐字相等，任一不符即抛 `XdbException`。因此跑完即代表零误差。

| 缓存策略 | 查询次数 | 总耗时 | 平摊 |
| --- | --- | --- | --- |
| file | 1,033,720 | 8 s | 7 μs/op |
| vectorIndex | 1,033,720 | 6 s | 6 μs/op |
| content | 1,033,720 | 0 s | 0 μs/op |

日志：[04-bench-v4.log](./04-bench-v4.log)

## 7. 已知问题（非阻塞）

1. `data/sample/ip.test.txt` 为旧版中文 region 格式，与当前英文 xdb 不匹配，用它跑 bench 会报 `failed search(1.0.0.0)`。属上游样例文件滞后，非代码缺陷；改用 `data/ipv4_source.txt` 通过。
2. 使用稀疏检出时，`sparse-checkout add` 会移除未纳入规则的已跟踪文件（xdb），需重新放置并复核哈希。

## 8. 复现命令

```
cd /Users/panzhuo/Documents/openclaw/ip2region/binding/java
JAVA_HOME=$(/usr/libexec/java_home -v 1.8) mvn -B clean package
JAVA_HOME=$(/usr/libexec/java_home -v 1.8) mvn -B test

cd ../../
J=$(/usr/libexec/java_home -v 1.8)/bin/java
$J -jar binding/java/target/ip2region-3.3.7.jar search
$J -jar binding/java/target/ip2region-3.3.7.jar bench \
   --db=data/ip2region_v4.xdb --src=data/ipv4_source.txt --cache-policy=vectorIndex
```
