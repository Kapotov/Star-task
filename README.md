#!/usr/bin/env python3

import sys, os
import requests
import json
import argparse
from datetime import datetime

# местоположение файла конфигурации
config_path = '/config_path'
# местоположение инициализированного репозитория Git
git_repo_path = '~/git/repo_path'
# Имя которе будет назначаться ветке для Pool Request
git_branch = datetime.now().strftime('%Y-%m-%d') + '-new-config'

# токен пользователя Github
github_token = 'ghp_xxxxxxxxxxxxxxxxxxxxx'
# URL для Pool Request
github_pr_url = 'https://api.github.com/repos/{owner}/{repo}/pulls'

# определение и получение параметра
parser = argparse.ArgumentParser(description='Send configuration file to Github')
parser.add_argument(
    'message',
    type=str,
    help='provide message for pull request'
    )
args = parser.parse_args()

# переходим в каталог уже настроенного git репозитория
os.chdir(git_repo_path)

# вытягиваем все изменения Git для измегания Merge Conflict
print ('git pull')
result = os.system('git pull')
print ('')

# создаем ветку репозитория 
print ('git checkout')
result = os.system('git checkout -b {}'.format(git_branch))
print ('')

# заменяем файлы конфигурации на конфигуарции с сервера, слэш указан для того чтобы обойти алиас cp='cp -i'
result = os.system('\cp -rf {} {}'.format(os.path.join(config_path,'*'),git_repo_path))

# добавляем новые файлы в репозиторий
print ('git add')
result = os.system('git add .')
print ('')

# коммитим файлы в репозиторий, для сообщения используем входой параметр message
print ('git commit')
commit_result = os.system('git commit -m "{}"'.format(args.message))
print ('')

if commit_result == 0:
    # отправляем ветку в удаленный репозиторий только если все хорошо, чтобы не сорить
    print ('git push')
    result = os.system('git push -fu origin {}'.format(git_branch))
    print ('')

# переключаемся в основную ветку 
print ('git checkout main')
result = os.system('git checkout main')
print ('')

# удаляем ветку для изменений
print ('git branch -d')
result = os.system('git branch -d {}'.format(git_branch))
print ('')

# если коммит завершился с ошибками, выходим после переключения ветки на главную и удаления ветки конфигурации
if commit_result > 0:
    if commit_result == 256: # нет измененных файлов
        print ('No changes detected\n')
    else:
        print ('Error detected during commit\n')
    sys.exit(0)

# создаем Pull Request на Github
headers_git={'Accept':'application/vnd.github.v3+json','Authorization':'token {}'.format(github_token)}
data_git={'base':'main','head':git_branch,'title':args.message}
r = requests.post(github_pr_url,data = json.dumps(data_git),headers = headers_git)

if (r.status_code == 201):
    print ('The pool request was created. Url: {}'.format(r.json()['html_url']))
else:
    print ('Error:')
    print (r)
    print (r.text)
