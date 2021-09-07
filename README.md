# Ansible LAB #4 - 把 ACI 中的 Fault Count 发送到 Webex Team 

<br><br>

## Lab 步骤

<br>

1. 在 ACI System > Faults 界面查看当前 Faults 信息。

![](../images/lab-ansible-4/lab-ansible-4-1.png)

<br><br>

2. 查看在 ansible 抓取 Fault 信息的脚本。(main.yml)
```yaml
  tasks:
    - name: "1] API Request - 收集 System Fault 概况"
      aci_rest:
        <<: *apic_info
        path:   "/api/node/class/faultSummary.json?query-target-filter=or(eq(faultSummary.severity,\"critical\"),eq(faultSummary.severity,\"major\"))&order-by=faultSummary.severity|desc"
        method: get
      register: faults_system
    
    - name: "2] (OPTIONAL) 把收集的信息导出到 JSON 文件中"
      copy:
        content: "{{ faults_system | to_nice_json}}"
        dest:    "./faults_system.json"
```
- 通过调用 REST API 中的 faultSummary class 来抓取 Faults 信息。
- 利用 query-target-filter 把特定严重等级的 Faults 抓取并排序。
- (OPTIONAL) 把收集的信息以 JSON 文件格式存储。此步骤是为了在 Lab 环境中更好的查看相关数据。

<br><br>

3. 执行 Playbook，查看收集到的信息。

```
ansible-playbook -i hosts main.yml
```

- 查看 faults_system.json 文件内容。

  ![](../images/lab-ansible-4/lab-ansible-4-2.png)

<br><br>

4. 把收集到的数据以 Webex 识别的格式进行转换。
- main.yml
```yaml
    - name: "3] 增加变量 - 消息标题"
      set_fact:
        add_title: "CHANGE_ME"

    - name: "4] 把数据转换为 Markdown 格式"
      template: 
        src:  "template/faults_report.j2"
        dest: "faults_report.md"
```
- "CHANGE_ME" 修改为自定义的标题名称。

<br>

- 以 template/faults_report.j2 文件把 faults_system 里面的数据转换为 Markdown 文件格式。
```jinja
---
# ACI Faults 通知 - {{ add_title }}

fault 总个数: **{{ faults_system.totalCount }}**

| Serverity | Type | Code | Domain | Cause |

{% for fault in faults_system.imdata %}
- **{{ fault.faultSummary.attributes.severity }}** | **{{ fault.faultSummary.attributes.type }}** | **{{ fault.faultSummary.attributes.code }}** | **{{ fault.faultSummary.attributes.domain }}** | `{{ fault.faultSummary.attributes.cause }}`
{% endfor %}
```

<br><br>

5. 执行 Playbook，查看生成的消息。
```
ansible-playbook -i hosts main.yml
```
<br>

- 查看 fault_report.md 文件内容。

    ![](../images/lab-ansible-4/lab-ansible-4-3.png)

<br><br>

6. 编写发送 Webex 消息的 Task 脚本。

```yaml
    - name: "5] 读取要发往 Webex 的消息的文件"
      debug:    msg="{{ lookup('file', 'faults_report.md') }}"
      register: faults_report
      no_log:   yes 

    - name: "6] 发送 Webex 消息"
      cisco_spark:
        recipient_type: roomId
        recipient_id:   "{{ roomID }}"
        message_type:   markdown
        personal_token: "{{ bot_token }}"
        message:        "{{ faults_report.msg }}"
```
- lookup('file', 文件名) 方法是把文件内容以文本的形式读取。

<br><br>

7. 在 Inventory 文件中修改 Webex Teams 联动时需要的信息。 
```
[aci:vars]
...
roomID=CHANGE_ME
bot_token=CHANGE_ME
```
- roomID 是接收消息的 Webex Teams Room ID。
- bot_token 是发送消息的 Webex Bot 信息。
- Webex Teams Room 和 Bot 需要先配置完成。


<br><br>

8. 执行 Playbook，并在 Webex Teams 中查看是否收到相应的消息。
```
ansible-playbook -i hosts main.yml
```
<br>


- 查看 fault_report.md 文件内容。

    ![](../images/lab-ansible-4/lab-ansible-4-4.png)
