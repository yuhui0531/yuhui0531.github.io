```python
# coding=utf-8
# author : Ronny
"""只能发送文字消息和简单的表情,不能群聊
存在相同好友名时会导致只能选择同名中的一个"""

import _thread
import logging
import os
import shutil
import socket
import time
import uuid

import itchat
from itchat.content import TEXT, RECORDING, PICTURE, VIDEO, ATTACHMENT
from prettytable import PrettyTable

from read_xml import parse_news_xml_str

FORMAT = '%(asctime)s | %(levelname)s | %(message)s'
fh = logging.FileHandler('wechat.log', encoding='utf-8')  # 创建一个文件流并设置编码utf8
logger = logging.getLogger()  # 获得一个logger对象，默认是root
logger.setLevel(logging.DEBUG)  # 设置最低等级debug
fm = logging.Formatter("%(asctime)s --- %(message)s")  # 设置日志格式
logger.addHandler(fh)  # 把文件流添加进来，流向写入到文件
fh.setFormatter(fm)  # 把文件流添加写入格式

top_name_list = ['都市夜归人', '乖乖', '范媛凤', '任超全', '郑清文', '3-1-2502春蕾', '松梅', '赵baimei(Lucy)']

HEADER_COUNT = 4
REMARK_NAME_PREFIX = 'RemarkName'
INDEX_PREFIX = 'Index'

REMARK_NAME_LIST = [REMARK_NAME_PREFIX + str(i) for i in range(1, HEADER_COUNT + 1)]
INDEX_LIST = [INDEX_PREFIX + str(i) for i in range(1, HEADER_COUNT + 1)]
TABLE_HEADERS = sum([[INDEX_LIST[i], REMARK_NAME_LIST[i]] for i in range(0, HEADER_COUNT)], [])


class WechatClient:

    def __init__(self):
        self.table_headers = TABLE_HEADERS
        self.table_headers_size = len(self.table_headers)
        self.pretty_table = self._prepare_pt()
        # 维护所有联系人聊天记录的大字典
        self.friends_message_dict = dict()
        # 最近联系人的列表
        self.recently_friends_list = list()
        self.recently_groups_list = list()
        # 最近联系人的列表的标注行用于选择
        self.recently_friends_line = ''
        self.recently_groups_line = ''
        # 所有联系人的列表
        self.all_friends_list = list()
        self.all_groups_list = list()
        # 锁，强制原子性操作
        self.a_lock = _thread.allocate_lock()
        self.b_lock = _thread.allocate_lock()
        self.c_lock = _thread.allocate_lock()

    def _prepare_pt(self):
        pretty_table = PrettyTable(self.table_headers)
        for p in REMARK_NAME_LIST:
            pretty_table.align[p] = 'l'
        return pretty_table

    @staticmethod
    def get_datetime_now():
        """获取当前时间并格式化"""
        return time.strftime('%Y-%m-%d %H:%M:%S')

    def _show_table(self, data):
        self.pretty_table.clear_rows()
        rows = []
        for index, friend in enumerate(data):
            if index > 0 and index * 2 % self.table_headers_size == 0:
                self.pretty_table.add_row(rows)
                rows.clear()
            rows.append(index + 1)
            rows.append(friend)
            if index == len(data) - 1:
                row_data_size = len(rows)
                if row_data_size < self.table_headers_size:
                    rows += [''] * (self.table_headers_size - row_data_size)
                self.pretty_table.add_row(rows)
        print(self.pretty_table)

    def update_recently_contact(self, contact, contact_type='friend'):
        self.a_lock.acquire()

        if contact_type == 'friend':
            contact_list = self.recently_friends_list
        else:
            contact_list = self.recently_groups_list

        if contact not in contact_list:
            contact_list.insert(0, contact)
        else:
            contact_list.remove(contact)
            contact_list.insert(0, contact)
        if len(contact_list) > 10:
            contact_list.pop()
        contact_line = ''
        for i, j in enumerate(contact_list):
            contact_line += f"{i + 1} :{j}; "

        if contact_type == 'friend':
            self.recently_friends_line = contact_line
        else:
            self.recently_groups_line = contact_line
        self.a_lock.release()

    def save_message_and_send_to_terminal(self, friend_name, line_message):
        self.b_lock.acquire()
        if friend_name not in self.friends_message_dict:
            self.friends_message_dict[friend_name] = [line_message]
        else:
            self.friends_message_dict[friend_name].append(line_message)
        # 发送到消息滚动屏幕，另一个终端
        _thread.start_new_thread(self._send_message_to_another_terminal, (line_message,))
        self.b_lock.release()

    @staticmethod
    def _send_message_to_another_terminal(message):
        """
        把消息通过套接字的形式发送出去，用于另一个终端的滚动打印，
        用于正在聊天时还能收到另外用户发来的消息。
        """
        try:
            client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            client.settimeout(0.01)
            client.connect(('127.0.0.1', 9527))
            try:
                message = message.encode('utf-8', 'ignore')
            except UnicodeEncodeError:
                logger.warning(f"unsupported message {message}")
                message = '不支持的消息'.encode('utf-8')
            client.send(message)
            client.close()
        except IOError:
            pass

    def _update_all_friends_list_and_line(self):
        print("登录成功，开始获取好友信息...")
        logger.info("登录成功，开始获取好友信息...")
        if not self.all_friends_list:
            all_friends = itchat.get_friends(update=True)
            temp_list = self._uniq([f['RemarkName'] or f['NickName'] for f in all_friends])
            difference = list(set(temp_list) - set(top_name_list))
            difference.sort()
            self.all_friends_list = top_name_list + difference
            how_many = len(self.all_friends_list)
            logger.info(f"成功获取到{how_many}个好友信息。")
            print(f"成功获取到{how_many}个好友信息。")

        if not self.all_groups_list:
            all_groups = itchat.get_chatrooms(update=True)
            self.all_groups_list = self._uniq([f['RemarkName'] or f['NickName'] for f in all_groups])
            how_many = len(self.all_groups_list)
            logger.info(f"成功获取到{how_many}个群信息。")
            print(f"成功获取到{how_many}个群信息。")

    @staticmethod
    def _uniq(friends):
        temp_list = []
        for i, f in enumerate(friends):
            if f not in temp_list:
                temp_list.append(f)
            else:
                temp_list.append(f + '_' + str(i))
        return temp_list

    def _get_contact(self, command):
        if command == 'l':
            return self.recently_friends_list[0]
        elif command == 'r':
            contacts = self.recently_friends_list
            t = '最近联系人'
        elif command == 'g':
            contacts = self.recently_groups_list
            t = '最近群聊'
        elif command == 'c':
            contacts = self.all_groups_list
            t = '总聊天群'
        else:
            contacts = self.all_friends_list
            t = '总联系人'

        if command in contacts:
            return command

        c = input(f'>>> {t}目录,输入名称或坐标: ')
        if c.isdigit():
            n = int(c)
            if n not in range(1, len(contacts) + 1):
                return contacts[0]
            else:
                return contacts[n - 1]
        else:
            if c in contacts:
                return c
            else:
                return contacts[0]

    @staticmethod
    def _get_prompt():
        # 根菜单输入
        prompt = ['', '==========根菜单==========',
                  '<1> l: 选择上一位联系人',
                  '<2> r: 展示最近联系人列表(最多十个)',
                  '<3> g: 展示最近群聊列表(最多十个)',
                  '<4> c: 展示所有的聊天群目录',
                  '<5>  : 其它的输入，直接展示联系人目录,若为联系人姓名或其在目录的坐标',
                  '']
        prompt = '\n'.join(prompt)
        prompt += '请输入你的选项: '
        return prompt

    def _choose_friend_and_chat(self):
        time.sleep(5)
        while True:
            command = input(self._get_prompt())
            # 根据输入选择用户
            if command == 'r':
                if not self.recently_friends_list:
                    continue
                print("\n-------最近联系人的目录-----")
                print(self.recently_friends_line)
            elif command == 'g':
                if not self.recently_groups_list:
                    continue
                print("\n-------最近联系人的目录-----")
                print(self.recently_groups_line)
            elif command == 'c':
                print('\n-------总聊天群目录-----')
                self._show_table(self.all_groups_list)
            elif command != 'l':
                print('\n-------总联系人的目录-----')
                self._show_table(self.all_friends_list)

            contact_name = self._get_contact(command)
            if not contact_name:
                continue

            # 1v1聊天
            print(f"\n******* {contact_name} *******")
            print('>> 输入 "b" 退出当前聊天 ，"enter" 发送消息或展示聊天记录... ')
            while True:
                try:
                    message = input(f"To {contact_name}: ")
                except UnicodeDecodeError:
                    continue

                # 返回根菜单， “退出聊天”
                if message == 'b':
                    break
                # 空白enter，展示与该用户的聊天记录
                elif message == '':
                    if contact_name in self.friends_message_dict.keys():
                        for mes in self.friends_message_dict[contact_name]:
                            print(mes)
                        print()
                else:
                    if command == 'c':
                        friend = itchat.search_chatrooms(name=contact_name)[0]
                    else:
                        friend = itchat.search_friends(name=contact_name)[0]
                    friend.send(message)
                    # 格式化发送的消息
                    line_message = f"{self.get_datetime_now()} To {contact_name}:  {message}"
                    # 更新最近联系人列表
                    self.update_recently_contact(contact_name)
                    # 处理消息，存到消息记录大字典，发送到滚动屏幕
                    self.save_message_and_send_to_terminal(contact_name, line_message)

    def run(self):
        """
        入口函数，
            启动一个where 1 的线程用于实时聊天
            主线程用于监听实时的消息
         """
        itchat.auto_login(hotReload=True, loginCallback=self._update_all_friends_list_and_line)
        _thread.start_new_thread(self._choose_friend_and_chat, ())
        itchat.run()


c = WechatClient()


@itchat.msg_register(TEXT)
def _receive_handle(msg):
    message = msg['Text']
    user = msg['User']
    is_xml = True
    if hasattr(user, 'RemarkName') and hasattr(user, 'NickName'):
        friend_name = user['RemarkName'] or user['NickName']
        is_xml = False
    elif hasattr(user, 'UserName') and user['UserName'] == 'filehelper':
        friend_name = 'filehelper'
        is_xml = False

    if is_xml:
        news_list = parse_news_xml_str(message)
        for item in news_list:
            line_message = f"{c.get_datetime_now()} From {item['publisher']}: " \
                           f"{item['pub_time']}发布新闻: " \
                           f"{item['title']} -- {item['digest']}"
            c.save_message_and_send_to_terminal(item['publisher'], line_message)
    else:
        # 更新最近联系人列表
        if friend_name == 'filehelper':
            logger.info(f'filehelper received TEXT message {message}')
            # 格式化收到的消息
            line_message = f"{c.get_datetime_now()} filehelper received: {message}"
        else:
            c.update_recently_contact(friend_name)
            line_message = f"{c.get_datetime_now()} From {friend_name}: {message}"
            logger.info(f'received TEXT message {message} from {friend_name}')
        # 添加消息到消息记录大字典，发送到滚动屏幕
        c.save_message_and_send_to_terminal(friend_name, line_message)


@itchat.msg_register([PICTURE, RECORDING, ATTACHMENT, VIDEO])
def _receive_image_handle(msg):
    filename = msg['FileName']
    message_type = msg['Type']
    user = msg['User']
    if hasattr(user, 'RemarkName') and hasattr(user, 'NickName'):
        friend_name = user['RemarkName'] or user['NickName']
    elif hasattr(user, 'UserName') and user['UserName'] == 'filehelper':
        friend_name = 'filehelper'
    else:
        friend_name = 'NA'
        logger.warning(f"unknown user info {user}")
    msg.download(filename)
    if not os.path.exists(friend_name):
        os.mkdir(friend_name)
    dest_file_path = os.path.join(friend_name, filename)
    if os.path.exists(dest_file_path):
        new_name = filename + '_' + uuid.uuid4().hex + '.' + filename.split('.')[-1]
        shutil.move(filename, new_name)
        filename = new_name

    shutil.move(filename, friend_name)
    if friend_name == 'filehelper':
        logger.info(f'filehelper received {message_type} message {filename}')
        line_message = f"{c.get_datetime_now()} filehelper received: {message_type} - {filename}"
    else:
        logger.info(f'received {message_type} message {filename} from {friend_name}')
        line_message = f"{c.get_datetime_now()} From {friend_name}: {message_type} - {filename}"
    c.save_message_and_send_to_terminal(friend_name, line_message)


@itchat.msg_register(TEXT, isGroupChat=True)
def _receive_group_message(msg):
    """
    监听群消息
    """
    user = msg['User']
    message = msg['Content']
    member_name = msg['ActualNickName']
    if hasattr(user, 'RemarkName') and hasattr(user, 'NickName'):
        group_name = user['RemarkName'] or user['NickName']
        c.update_recently_contact(group_name, 'group')
    else:
        group_name = 'NA'
        logger.warning(f"unknown user info {user}")

    logger.info(f'received TEXT message {message} from {member_name} of GROUP {group_name}')
    # 格式化收到的消息
    line_message = f"{c.get_datetime_now()} {group_name}<{member_name}>: {message}"
    # 添加消息到消息记录大字典，发送到滚动屏幕
    c.save_message_and_send_to_terminal(group_name, line_message)


if __name__ == '__main__':
    c.run()

```

