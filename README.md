# sharedlib-multi-repo-jenkinsfile
Bu repo içinde sadece Jenkinsfile barındıracak. Böylece vs code ile pipeline kodunu güzel güzel yazacağım. Sonra ana proje çalıştığında verilen parametrelere göre tüm repoları çekip, derleyip, test, yayın işlerini buradaki Jenkinsfile içindeki kodlarla sharedlib-multi-repo havuzundaki sınıflar üstünden yürütecek.

# Jenkinsfile
Bu dosya baş rol oyuncumuz. Jenkins bu dosya üstünden pipeline çalıştıracak.