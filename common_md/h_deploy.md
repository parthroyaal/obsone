
````bash
git clone git@github.com:parth-royal/linked-blog-starter-md.git

wget https://github.com/zoni/obsidian-export/releases/download/v22.11.0/obsidian-export_Linux-x86_64.bin

chmod +x obsidian-export_Linux-x86_64.bin



mkdir temp_blog


common_md --> NextPrj

rm -rf temp_blog/common_md && mkdir temp_blog/common_md && rm -rf temp_md/*

cp -r 2024obs/aug20 temp_md/. # copy obsidan date frolder md files to temp_md

/home/acer/Documents/2024/0_obsidianX/bk/bk1
cp -r /home/acer/Documents/2024/0_obsidianX/bk/bk1 ~/Desktop/obscloud/temp_md/.


obsidian-export source destination
./obsidian-export_Linux-x86_64.bin ./temp_md/bk1 temp_blog/common_md

rm -r linked-blog-starter/common_md/
mv temp_blog/common_md/ linked-blog-starter/.

npm run dev 

.export-ignore
.gitignore
````
