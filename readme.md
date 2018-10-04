# Blog

[如何備份、還原 GitLab 服務][blog]

> GitLab Docker-Compose.yml 採用 [sameersbn][url] 製作的版本

[blog]: https://dotblogs.com.tw/artblog/2018/10/04/how-to-backup-restore-docker-gitlab
[url]: https://github.com/sameersbn/docker-gitlab

# 建立 GitLab 服務

```bash
# 建立 Volume
docker volume create gitlab_data
docker volume create gitlab_backup
docker volume create postgres_data
docker volume create redis_data

# 建立 Gitlab 服務
docker-compose up
```

# 備份 GitLab

```bash
# Create Backup
docker-compose exec gitlab su -c "bundle exec rake gitlab:backup:create" git

# Restore Backup
docker-compose exec gitlab supervisorctl stop unicorn
docker-compose exec gitlab supervisorctl stop sidekiq

# 列出有哪些備份可還原
docker-compose exec gitlab su -c "bundle exec rake gitlab:backup:restore" git
# 指定 TimeStamp 還原
docker-compose exec gitlab su -c "bundle exec rake gitlab:backup:restore BACKUP=1538620541_2018_10_04_11.3.0" git

docker-compose exec gitlab supervisorctl start sidekiq
docker-compose exec gitlab supervisorctl start unicorn
```

# 如何將 docker volume 以實體檔案的方式備份、還原

## 將 docker volume 備份為實體檔案

```bash
# 建立 container 用來取得 gitlab_backup volume 的備份檔案
docker run -d --name mybackup -v gitlab_backup:/volume alpine ping 127.0.0.1

# 使用 tar 壓縮整個 volume 目錄
docker exec -it mybackup tar -cjf /data.tar.bz2 -C /volume ./

# 用 docker inspect或者是 docker ps 查詢目前 container 的 Id
docker inspect --format="{{.Id}}" mybackup

# 透過 container Id 將 container 打包成一個新的images：my-volume-backup
docker commit -p e0a4366ae143 my-volume-backup

# 將images導出為實體檔案
docker save -o my-volume-backup.tar my-volume-backup
```

## 將實體檔案還原為 docker volume

```bash
# 建立 Volume
docker volume create gitlab_data
docker volume create gitlab_backup
docker volume create postgres_data
docker volume create redis_data

# 將 Gitlab 服務架起來
docker-compose up

# 將實體檔案還原為 Image
docker load -i my-volume-backup.tar

# 將 Image 裡面的檔案解壓縮到 Docker Volume
docker run  --rm  -v gitlab_backup:/volume my-volume-backup sh -c "rm -rf /volume/* /volume/..?* /volume/.[!.]* ; tar -C /volume/ -xjf  /data.tar.bz2 ;"

# 把腳本建立的 GitLab 名稱改名為 gitlab (這一步省略的話接下來要自己替換掉 container name)
docker rename docker-gitlab_gitlab_1 gitlab

# 記得這個時候要確認 GitLab 是 start 的，否則會出現錯誤訊息說找不到 container
# 先確認有哪些備份檔可還原
docker-compose exec gitlab su -c "bundle exec rake gitlab:backup:restore" git

# 指定 TimeStamp 還原
docker-compose exec gitlab supervisorctl stop unicorn
docker-compose exec gitlab supervisorctl stop sidekiq
docker-compose exec gitlab su -c "bundle exec rake gitlab:backup:restore BACKUP=1538620541_2018_10_04_11.3.0" git
docker-compose exec gitlab supervisorctl start sidekiq
docker-compose exec gitlab supervisorctl start unicorn
```
