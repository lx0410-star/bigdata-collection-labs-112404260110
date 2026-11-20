# 实验三：QQ邮箱采集（Selenium版）

Selenium与[Playwright版](实验指导书-Playwright版.md)二选一，完成其中一版即可。

使用Selenium完成四阶段。**正文必做，附件选做。**

| 阶段 | 课时 | 完成效果 |
|---|---|---|
| 1. 扫码登录 | 第1—2课时 | 未扫码时等待；扫码后进入收件箱 |
| 2. 采集和保存 | 第3—4课时 | 采集10封邮件，用pandas保存并核对CSV |
| 3. 滚动采集 | 第5—6课时 | 连续采集3批，按邮件ID去重 |
| 4. 详情正文 | 第7—8课时 | 逐封读取正文，返回后继续；附件下载选做 |

## 使用方法

1. 在JupyterLab的 `workspace/lab03/` 中，自己新建 `qqmail_lab03.ipynb`。
2. 四阶段写在这一个Notebook中，按“单元1、2、3……”依次添加代码。
3. **补全本阶段所有 `____` 后再运行。** 空白包括方法名、选择器和参数。
4. 后面直接使用前面的变量和函数，不另换一套名称。
5. 下次课若重启了内核，先执行前面已完成的单元，再继续。

四阶段填写同一份[实验报告](实验报告-Selenium版.md)，最后统一提交一次GitHub。

## 第一阶段：扫码登录（第1—2课时）

### 目标与准备

**目标：** 浏览器打开QQ邮箱，扫码后输出“登录成功”，并进入收件箱。

未安装visual的同学，先按[可视化安装说明](可视化环境安装说明.md)安装：

1. 打开Docker Desktop和原课程环境。
2. 按教师发放的增量包说明，运行 `install-visual.bat`。
3. 运行 `start-course-visual.bat`，确认JupyterLab和浏览器画面都打开。

`lab03` 要与已有的 `L07_L08_visual` 同级。不领取完整采集程序，也不复制额外辅助脚本。

### 单元1：导入工具，准备浏览器

环境配置用于连接Docker中的显示服务，不要求背诵路径。补全创建浏览器的方法名。

```python
import sys
from pathlib import Path
import pandas as pd
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.common.by import By
from selenium.webdriver.common.action_chains import ActionChains
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.common.exceptions import StaleElementReferenceException

# 建立后面三个阶段共用的保存目录。
PRIVATE = Path.cwd() / "private_data"
PRIVATE.mkdir(exist_ok=True)

# course_visual属于已安装的visual环境，不需要另发文件。
sys.path.insert(0, str(Path.cwd().parent / "L07_L08_visual"))
import course_visual
course_visual.prepare()

# 连接课程Docker中的浏览器和配套驱动。
options = webdriver.ChromeOptions()
options.binary_location = str(next(
    Path("/ms-playwright").glob("chromium-*/chrome-linux*/chrome")
))
options.add_argument("--no-sandbox")
options.add_argument("--disable-dev-shm-usage")
options.add_argument("--window-size=1200,800")
driver_path = Path.home() / ".cache/selenium/chromedriver/linux64/151.0.7922.138/chromedriver"
wd = webdriver.____(service=Service(str(driver_path)), options=options)
# 页面更新时，旧元素可能失效。等待过程中遇到此异常，就重新尝试。
wait = WebDriverWait(wd, 20, ignored_exceptions=(StaleElementReferenceException,))
```

提示：创建Chromium浏览器用 `webdriver.Chrome()`。缺少工具或驱动时，请教师检查课程环境。

### 单元2：打开邮箱，扫码并等待

```python
# ① 补“打开网页”的方法名。
wd.____("https://mail.qq.com/")
print("请在浏览器画面中扫码", flush=True)

# ② 本人扫码。查看登录后的页面，定位左侧“收件箱”。
# 补By中的定位方式，以及该控件的选择器。
inbox = (By.____, "____")

# ③ 最长等待600秒，控件可点击后立即继续。
# 补时间和“元素可点击”的条件名。
WebDriverWait(wd, ____).until(EC.____(inbox))
print("登录成功", flush=True)

# ④ *inbox将定位方式和选择器传给find_element。补点击方法。
wd.find_element(*inbox).____()
```

| 关键函数 | 用途 |
|---|---|
| `get(url)` | 打开网页 |
| `find_element(By.CSS_SELECTOR, "选择器")` | 用CSS定位一个元素 |
| `find_element(By.XPATH, "路径")` | 用XPath定位一个元素 |
| `WebDriverWait(...).until(条件)` | 等待条件满足 |
| `EC.element_to_be_clickable(定位信息)` | 检查元素能否点击 |
| `click()` | 点击 |

**完成证据：** 补全后的定位、等待、点击代码，以及“登录成功”和收件箱截图。遮挡私人信息，解释等待条件。

## 第二阶段：采集10封邮件并保存（第3—4课时）

### 本阶段目标

采集发件人、邮箱、主题、时间。用pandas保存CSV，回读后有10条记录，每条4项。

### 单元3：自己找选择器

只给两条线索，其余请在开发者工具中查找：

| 内容 | 定位线索 |
|---|---|
| 发件人姓名 | `.cmp-account-nick` |
| 广告聚合入口标记 | `.gg-unread-count`，不是单封邮件，应跳过 |

先找到一封邮件的父容器，再找里面的主题和时间。悬停发件人后，查看邮箱卡片。卡片要在整个页面中查找。

```python
ROW_CSS = "____"           # 一条邮件项目
SENDER_CSS = ".cmp-account-nick"
AD_CSS = ".gg-unread-count"
SUBJECT_CSS = "____"       # 行内主题
TIME_CSS = "____"          # 行内时间
HOVER_CSS = "____"         # 行内发件人悬停区域
EMAIL_CSS = "____"         # 页面中的邮箱卡片字段
```

### 单元4：定义列表解析函数

这些函数会继续用于第三、四阶段。每封邮件用一个字典保存。

```python
def get_ids():
    # 整批ID都在等待函数内读取。遇到旧元素失效，就重读整批。
    def read_current_ids(d):
        rows = d.find_elements(By.CSS_SELECTOR, ROW_CSS)
        if not rows:
            return False  # 列表还没出现，继续等待。
        ids = [row.get_attribute("data-mailid") for row in rows]
        if not all(ids):
            raise ValueError("发现没有ID的列表项，请检查定位")
        return ids
    return wait.until(read_current_ids)


def get_row(mail_id):
    return wd.find_element(
        By.CSS_SELECTOR, f'{ROW_CSS}[data-mailid="{mail_id}"]'
    )


def read_list_mail(mail_id):
    row = get_row(mail_id)
    # 补“查找多个元素”的方法。存在广告标记时跳过。
    if row.____(By.CSS_SELECTOR, AD_CSS):
        return None

    sender = row.find_element(By.CSS_SELECTOR, SENDER_CSS).____
    subject = row.find_element(By.CSS_SELECTOR, SUBJECT_CSS).____
    time_text = row.find_element(By.CSS_SELECTOR, TIME_CSS).____

    # 先移开鼠标，等待上一张卡片消失。
    time_node = row.find_element(By.CSS_SELECTOR, TIME_CSS)
    ActionChains(wd).move_to_element(time_node).____()
    wait.until(EC.____((By.CSS_SELECTOR, EMAIL_CSS)))

    # 再悬停当前发件人，等卡片出现后读取邮箱。
    sender_node = row.find_element(By.CSS_SELECTOR, HOVER_CSS)
    ActionChains(wd).move_to_element(sender_node).____()
    email_node = wait.until(EC.____((By.CSS_SELECTOR, EMAIL_CSS)))
    email = email_node.____

    return {"邮件ID": mail_id, "发件人": sender,
            "邮箱": email, "主题": subject, "时间": time_text}
```

| 关键代码 | 用途 |
|---|---|
| `find_elements(...)` | 查找多个元素；找不到时返回空列表 |
| `.text` | 读取文字 |
| `get_attribute(...)` | 读取属性 |
| `move_to_element(...).perform()` | 鼠标悬停 |
| `EC.invisibility_of_element_located(...)` | 等待消失 |
| `EC.visibility_of_element_located(...)` | 等待显示 |

### 单元5：采集10封

```python
basic_records = []
for mail_id in get_ids():
    record = read_list_mail(mail_id)
    if record is None:
        continue
    basic_records.____(record)
    if len(basic_records) == ____:
        break
print("成功采集：", len(basic_records))
```

补列表的 `append()` 和目标条数。先核对一封，再运行完整循环。

### 单元6：用pandas保存和回读

定义一次，后三种结果都用这两个函数保存和回读。

```python
def save_records(records, filename, columns):
    df = pd.____(records, columns=columns)
    path = PRIVATE / filename
    df.____(path, index=____, encoding="utf-8-sig")
    return path


def read_records(path):
    # dtype=str按文字读取；空字段保留为空字符串。
    return pd.____(path, dtype=str, keep_default_na=False)


BASIC_COLUMNS = ["发件人", "邮箱", "主题", "时间"]
basic_path = save_records(basic_records, "mail_basic.csv", BASIC_COLUMNS)
basic_df = read_records(basic_path)
print("CSV记录数：", len(basic_df))
basic_df.head(10)
```

| 关键代码 | 用途 |
|---|---|
| `pd.DataFrame(记录, columns=列名)` | 将字典列表变成表格 |
| `df.to_csv(路径, index=False, encoding="utf-8-sig")` | 保存CSV，不多写一列行号 |
| `pd.read_csv(路径, ...)` | 回读CSV |
| `len(df)`、`df.head(10)` | 查看条数和前10条 |

**完成证据：** 字段提取、悬停等待、pandas保存和回读截图。CSV应为10行数据、4列。抽查第一条、中间一条和最后一条，与网页核对。

## 第三阶段：滚动采集与ID去重（第5—6课时）

### 本阶段目标

连续采集3批邮件，保存5列CSV。记录数等于唯一ID数，重复采集已有ID时新增0条。

### 1. 滚动代码是谁执行的

`wd.execute_script()` 是 **Selenium的方法**。引号内写的是 **JavaScript**，由浏览器执行。

Selenium负责把指令送到网页。JavaScript负责读取或修改滚动容器的属性。

| 属性 | 含义 |
|---|---|
| `scrollTop` | 已向下滚动的距离，可读取或修改 |
| `clientHeight` | 容器里当前可见的一屏高度 |
| `scrollHeight` | 整个内容的高度 |

例如 `wd.execute_script("return arguments[0].scrollTop", box)`：

- Python把 `box` 传入，JavaScript用 `arguments[0]` 取得它。
- `return` 把读到的位置送回Python。

写入时，用 `arguments[1]` 接收第二个参数，也就是目标位置。一屏向下移动通常为 `top + height`。末尾最大位置为 `scrollHeight - clientHeight`。

### 单元7：补全批次解析和滚动循环

自己找出邮件列表的滚动容器，补 `SCROLL_CSS`。其余继续使用前面的函数。

```python
SCROLL_CSS = "____"


def get_box():
    return wd.find_element(By.CSS_SELECTOR, SCROLL_CSS)


def parse_batch(mail_ids, seen_ids, include_details=False, max_mails=0):
    result = []
    for mail_id in mail_ids:
        if max_mails and len(seen_ids) >= max_mails:
            break
        if ____:  # 判断这个ID是否已在集合中
            continue
        record = read_list_mail(mail_id)
        if record is None:
            continue
        if include_details:
            # read_detail在第四阶段定义；本阶段False，不执行这里。
            record.update(read_detail(mail_id))
        result.____(record)
        seen_ids.____(mail_id)
    return result


def collect_batches(include_details=False, max_mails=0):
    # 每次开始这一轮采集，都回到顶部，并新建本轮集合。
    wd.find_element(*inbox).click()
    wait.until(EC.presence_of_element_located((By.CSS_SELECTOR, ROW_CSS)))
    wd.execute_script("arguments[0].scrollTop = 0", get_box())
    records, seen_ids = [], set()

    for batch in range(____):  # 本实验共3批
        top = wd.execute_script("return arguments[0].scrollTop", get_box())
        result = parse_batch(get_ids(), seen_ids, include_details, max_mails)
        records.extend(result)
        print(f"第{batch + 1}批，新增{len(result)}，累计{len(seen_ids)}")
        if batch == 2 or (max_mails and len(seen_ids) >= max_mails):
            break

        # 定位和悬停可能自动滚动，所以仍按本批开始位置计算。
        box = get_box()
        height = wd.execute_script("return arguments[0].clientHeight", box)
        full_height = wd.execute_script("return arguments[0].scrollHeight", box)
        next_top = min(____, full_height - height)
        if next_top <= top:
            print("已到列表末尾")
            break
        old_ids = get_ids()
        wd.execute_script("arguments[0].scrollTop = arguments[1]", box, next_top)
        wait.until(lambda d: ____ != old_ids)  # 重新取ID列表，等待内容改变
    return records, seen_ids
```

`include_details=False` 表示只采列表。第四阶段补好正文函数后，改为 `True`。`max_mails=0` 表示不另设封数上限。

### 单元8：执行3批，复用pandas保存函数

```python
PAGE_COLUMNS = ["邮件ID"] + BASIC_COLUMNS
page_records, seen_ids = ____()
page_path = ____(page_records, "mail_pages.csv", PAGE_COLUMNS)
page_df = ____(page_path)
print("CSV记录数：", len(page_df))
print("唯一ID数：", page_df["邮件ID"].____())
print("重复ID数：", page_df["邮件ID"].duplicated().sum())
```

`nunique()` 统计不同ID数，`duplicated().sum()` 统计重复ID数。3批条数以实际结果为准，不固定每批20封。

### 单元9：重放已采集的ID，验证去重

```python
replay_ids = [r["邮件ID"] for r in page_records[:5]]
assert replay_ids, "先成功采集一些邮件，再做重复验证"
before = page_path.read_bytes()
replayed = ____(replay_ids, seen_ids)
print("重复采集新增：", len(replayed))
____(page_records + replayed, "mail_pages.csv", PAGE_COLUMNS)
print("CSV内容未变：", before == page_path.read_bytes())
```

这里使用同一个 `seen_ids`，再次处理真实采集过的ID。不是关闭程序后重新覆盖文件。

**完成证据：** 滚动和去重代码、3批日志、CSV记录数与唯一ID数、重复采集新增0条的截图。

## 第四阶段：详情正文（第7—8课时）

### 本阶段目标

读取详情正文并返回列表，继续处理下一封。保留前三阶段代码，只追加下面的单元。

### 单元10：补全正文处理函数

自己定位主题、正文和“返回”控件。主题选择器继续使用单元3的 `SUBJECT_CSS`。

```python
BODY_CSS = "____"
BACK = (By.____, "____")
DOWNLOAD_ATTACHMENTS = False  # 必做先保持False；选做补全后才开启。


def read_detail(mail_id):
    top = wd.execute_script("return arguments[0].scrollTop", get_box())
    row = get_row(mail_id)
    row.find_element(By.CSS_SELECTOR, SUBJECT_CSS).____()
    try:
        body_node = wait.until(EC.____((By.CSS_SELECTOR, BODY_CSS)))
        body = body_node.____("innerText").strip()
        files = ""
        if DOWNLOAD_ATTACHMENTS:
            # 可选函数在单元13定义，关闭时不会调用。
            files = download_attachments(mail_id)
        return {"正文": body, "附件文件": files}
    finally:
        # 无论正文处理是否成功，都尝试返回列表。
        wd.find_element(*BACK).____()
        wait.until(EC.presence_of_element_located((By.CSS_SELECTOR, SCROLL_CSS)))
        wd.execute_script("arguments[0].scrollTop = arguments[1]", get_box(), ____)
        wait.until(EC.presence_of_element_located(
            (By.CSS_SELECTOR, f'{ROW_CSS}[data-mailid="{mail_id}"]')
        ))
```

`get_attribute("innerText")` 读取正文的可见文字。`finally` 中的代码负责返回列表、恢复位置。

### 单元11：先试3封

```python
DETAIL_COLUMNS = PAGE_COLUMNS + ["正文", "附件文件"]

# 先验证3封：进入详情、读正文、返回后继续。
trial_records, _ = ____(include_details=____, max_mails=____)
trial_path = ____(trial_records, "mail_details_trial.csv", DETAIL_COLUMNS)
trial_df = ____(trial_path)
print(trial_df[["邮件ID", "正文"]].assign(正文长度=trial_df["正文"].str.____()))
```

核对3封正文和返回操作。确认无误后，再运行下一个单元。

### 单元12：正式采集3批并保存

```python
detail_records, _ = ____(include_details=____)
detail_path = ____(detail_records, "mail_details.csv", DETAIL_COLUMNS)
detail_df = ____(detail_path)
print("正文CSV记录数：", len(detail_df))
print("重复ID数：", detail_df["邮件ID"].____().sum())
```

试采和正式采集会各自新建去重集合。正文中的逗号、引号和换行由pandas处理。不要按文本行数统计邮件数。

抽查3封，确认正文与详情页一致，不是列表摘要。纯图片邮件如实注明，不要求OCR。

### 单元13：选做附件下载框架

未选做时跳过本单元，保留 `DOWNLOAD_ATTACHMENTS=False`。选做时补全函数，再执行本单元末尾的3封试采。不必重新执行整个正文采集循环。

```python
ATTACH_CSS = "____"        # 附件卡片
DOWNLOAD = (By.____, "____")  # 卡片内“下载”控件


def download_attachments(mail_id):
    cards = wd.find_elements(By.CSS_SELECTOR, ATTACH_CSS)
    if not cards:
        return ""
    folder = PRIVATE / "attachments" / mail_id
    folder.mkdir(parents=True, exist_ok=True)
    wd.execute_cdp_cmd("Browser.setDownloadBehavior", {
        "behavior": "allow", "downloadPath": str(folder.resolve())
    })
    files = []
    for card in cards:
        before = set(folder.iterdir())
        card.find_element(*DOWNLOAD).____()

        def completed(d):
            added = set(folder.iterdir()) - before
            if added and all(p.is_file() and p.suffix != ".crdownload"
                             and p.stat().st_size > 0 for p in added):
                return added
            return False

        added = WebDriverWait(wd, 90).until(completed)
        files.extend(str(p.relative_to(PRIVATE)) for p in added)
    return "\n".join(files)


# 只验证3封；不会覆盖上面的正式正文CSV。
DOWNLOAD_ATTACHMENTS = True
attachment_records, _ = collect_batches(include_details=True, max_mails=3)
attachment_path = save_records(
    attachment_records, "mail_attachments_trial.csv", DETAIL_COLUMNS
)
print(read_records(attachment_path)[["邮件ID", "附件文件"]])
DOWNLOAD_ATTACHMENTS = False
```

`execute_cdp_cmd()` 设置Chromium下载目录；`iterdir()` 查看目录；`stat().st_size` 读取文件大小。下载中的文件通常以 `.crdownload` 结尾。

选择无敏感内容的练习附件，不要求发票。没有附件时，正文照常保存。

**完成证据：** 正文函数、3封核对样本、正文长度、正式CSV条数，以及返回后继续处理的截图。选做时另附文件名和大小。

## 最后统一提交

1. 提交自己补全的 `qqmail_lab03.ipynb`，四阶段单元按顺序保留。
2. 填写 `实验报告-Selenium版.md`。三份正式原始CSV保留在本地 `private_data/`。
3. 三份脱敏CSV放入 `data/`，文件名加 `_public`；截图放入 `images/`。
4. 四阶段结束后，将整理好的 `lab03` 一次上传到原GitHub仓库，再向学习通提交原仓库链接。

上传前检查Notebook输出、截图和正文。不要上传原始邮件、私人附件、Cookie、登录态或密码。

下课先保存Notebook，再运行 `wd.quit()`，最后运行 `stop-course-visual.bat`。下次课按顺序执行已补全的单元。

参考：[Selenium执行脚本](https://www.selenium.dev/selenium/docs/api/py/selenium_webdriver_remote/selenium.webdriver.remote.webdriver.html)、[pandas保存CSV](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.to_csv.html)、[pandas读取CSV](https://pandas.pydata.org/docs/reference/api/pandas.read_csv.html)。
