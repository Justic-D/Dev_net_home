### Домашнее задание к занятию "2.4 Инструменты Git"

#### 1. Найдите полный хеш и комментарий коммита, хеш которого начинается на aefea.

Решение  
git show aefea

**Ответ**  
aefead2207ef7e2aa5dc81a34aedf0cad4c32545  
Update CHANGELOG.md

#### 2. Какому тегу соответствует коммит 85024d3?

Решение  
git show 85024

**Ответ**  
commit 85024d3100126de36331c6982bfaac02cdab9e76 (tag: v0.12.23)

#### 3. Сколько родителей у коммита b8d720? Напишите их хеши.

Решение  
git show --pretty=format:' %P' b8d720

**Ответ**  
56cd7859e05c36c06b56d013b55a252d0bb7e158  
9ea88f22fc6269854151c571162c5bcf958bee2b

#### 4. Перечислите хеши и комментарии всех коммитов которые были сделаны между тегами v0.12.23 и v0.12.24.

Решение  
git log  v0.12.23..v0.12.24  --oneline

**Ответ**  
33ff1c03b (tag: v0.12.24) v0.12.24  
b14b74c49 [Website] vmc provider links  
3f235065b Update CHANGELOG.md  
6ae64e247 registry: Fix panic when server is unreachable  
5c619ca1b website: Remove links to the getting started guide's old location  
06275647e Update CHANGELOG.md  
d5f9411f5 command: Fix bug when using terraform login on Windows  
4b6d06cc5 Update CHANGELOG.md  
dd01a3507 Update CHANGELOG.md  
225466bc3 Cleanup after v0.12.23 release  

#### 5. Найдите коммит в котором была создана функция func providerSource, ее определение в коде выглядит так func providerSource(...) (вместо троеточего перечислены аргументы).

Решение  
git log -S'func providerSource(' --oneline

**Ответ**  
8c928e835 main: Consult local directories as potential mirrors of providers

#### 6. Найдите все коммиты в которых была изменена функция globalPluginDirs.

Решение  
git grep 'func globalPluginDirs'  
git log -L :'func globalPluginDirs':plugins.go --oneline  

**Ответ**  
commit 8364383c3 - создана  
commit 66ebff90c - изменена  
commit 41ab0aef7 - изменена  
commit 52dbf9483 - изменена  
commit 78b122055 - изменена  

#### 7. Кто автор функции synchronizedWriters?

Решение  

git log -S'func synchronizedWriters(‘ --pretty=format:'%h - %an %ae'

bdfea50cc - James Bardin j.bardin@gmail.com  
5ac311e2a - Martin Atkins mart@degeneration.co.uk  

git show bdfea50cc - удалена  
git show 5ac311e2a - создана  

**Ответ**  
Author: Martin Atkins <mart@degeneration.co.uk>