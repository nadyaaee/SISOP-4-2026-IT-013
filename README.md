
# SISOP-4-2026-IT-013
Laporan Resmi Praktikum Sistem Operasi Modul 2

## Penulis
Nadya Putri Agustin \
5027251013
## Soal 1

### `kenz_rescue.c`
```
#define FUSE_USE_VERSION 28

#include <fuse.h>
#include <stdio.h>
#include <string.h>
#include <errno.h>
#include <fcntl.h>
#include <dirent.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/stat.h>

static char source_dir[1024];

static void build_real_path(char fpath[1024], const char *path)
{
    sprintf(fpath, "%s%s", source_dir, path);
}

static void build_tujuan_content(char *result)
{
    result[0] = '\0';

    char gabungan[2048] = "";

    for (int i = 1; i <= 7; i++) {
        char path[1024];
        snprintf(path, sizeof(path), "%s/%d.txt", source_dir, i);

        FILE *fp = fopen(path, "r");
        if (!fp)
            continue;

        char line[1024];

        while (fgets(line, sizeof(line), fp)) {
            if (strncmp(line, "KOORD:", 6) == 0) {
                char *fragment = line + 6;

                while (*fragment == ' ')
                    fragment++;

                fragment[strcspn(fragment, "\n")] = 0;

                strcat(gabungan, fragment);
            }
        }

        fclose(fp);
    }

    sprintf(result, "Tujuan Mas Amba: %s\n", gabungan);
}

static int kenz_getattr(const char *path, struct stat *stbuf)
{
    int res;
    char fpath[1024];

    memset(stbuf, 0, sizeof(struct stat));

    if (strcmp(path, "/") == 0) {
        stbuf->st_mode = S_IFDIR | 0755;
        stbuf->st_nlink = 2;
        return 0;
    }

    if (strcmp(path, "/tujuan.txt") == 0) {
        char content[2048];
        build_tujuan_content(content);

        stbuf->st_mode = S_IFREG | 0444;
        stbuf->st_nlink = 1;
        stbuf->st_size = strlen(content);

        return 0;
    }

    build_real_path(fpath, path);

    res = lstat(fpath, stbuf);

    if (res == -1)
        return -errno;

    return 0;
}

static int kenz_readdir(const char *path,
                        void *buf,
                        fuse_fill_dir_t filler,
                        off_t offset,
                        struct fuse_file_info *fi)
{
    DIR *dp;
    struct dirent *de;

    char fpath[1024];

    (void) offset;
    (void) fi;

    if (strcmp(path, "/") != 0)
        return -ENOENT;

    filler(buf, ".", NULL, 0);
    filler(buf, "..", NULL, 0);

    sprintf(fpath, "%s", source_dir);

    dp = opendir(fpath);

    if (dp == NULL)
        return -errno;

    while ((de = readdir(dp)) != NULL) {
        struct stat st;

        if (strcmp(de->d_name, ".") == 0 || strcmp(de->d_name, "..") == 0)
            continue;

        if (strstr(de->d_name, ".zip") != NULL)
            continue;

        memset(&st, 0, sizeof(st));

        st.st_ino = de->d_ino;
        st.st_mode = de->d_type << 12;

        filler(buf, de->d_name, &st, 0);
    }

    closedir(dp);

    filler(buf, "tujuan.txt", NULL, 0);

    return 0;
}

static int kenz_open(const char *path, struct fuse_file_info *fi)
{
    char fpath[1024];

    (void) fi;

    if (strcmp(path, "/tujuan.txt") == 0)
        return 0;

    build_real_path(fpath, path);

    int fd = open(fpath, O_RDONLY);

    if (fd == -1)
        return -errno;

    close(fd);

    return 0;
}

static int kenz_read(const char *path,
                     char *buf,
                     size_t size,
                     off_t offset,
                     struct fuse_file_info *fi)
{
    (void) fi;

    if (strcmp(path, "/tujuan.txt") == 0) {
        char content[2048];

        build_tujuan_content(content);

        size_t len = strlen(content);

        if (offset < len) {
            if (offset + size > len)
                size = len - offset;

            memcpy(buf, content + offset, size);
        } else {
            size = 0;
        }

        return size;
    }

    char fpath[1024];

    build_real_path(fpath, path);

    int fd = open(fpath, O_RDONLY);

    if (fd == -1)
        return -errno;

    int res = pread(fd, buf, size, offset);

    if (res == -1)
        res = -errno;

    close(fd);

    return res;
}

static struct fuse_operations kenz_oper = {
    .getattr = kenz_getattr,
    .readdir = kenz_readdir,
    .open = kenz_open,
    .read = kenz_read,
};

int main(int argc, char *argv[])
{
    if (argc < 3) {
        fprintf(stderr, "Usage: %s <source_dir> <mount_dir>\n", argv[0]);
        return 1;
    }

    realpath(argv[1], source_dir);

    char *fuse_argv[2];
    fuse_argv[0] = argv[0];
    fuse_argv[1] = argv[2];

    int fuse_argc = 2;

    umask(0);

    return fuse_main(fuse_argc, fuse_argv, &kenz_oper, NULL);
}
```


### Header Library
```
#define FUSE_USE_VERSION 28
```
Program menggunakan FUSE versi 28.
```
#include <fuse.h>
#include <stdio.h>
#include <string.h>
#include <errno.h>
#include <fcntl.h>
#include <dirent.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/stat.h>
```
### Variabel Global
```
static char source_dir[1024];
```
Variabel ini menyimpan lokasi folder asli yang akan ditampilkan lewat mount FUSE.

### Fungsi `build_real_path`
```
static void build_real_path(...)
```
Fungsi ini menggabungkan `source_dir` dengan path dari FUSE agar program tahu lokasi file aslinya.

Jadi kalau user buka `/1.txt` di mount folder, program tahu file aslinya ada di `amba_files/1.txt`

### Fungsi `build_tujuan_content`
```
static void build_tujuan_content(char *result)
```
Fungsi ini membuat isi file virtual `tujuan.txt` dari gabungan data `KOORD:` pada file `1.txt` sampai `7.txt`.

Contoh isi `tujuan.txt` nanti:
```
Tujuan Mas Amba: koordinatgabungan
```

### Fungsi `kenz_getattr`
```
static int kenz_getattr(const char *path, struct stat *stbuf)
```
Fungsi ini dipanggil saat sistem ingin tahu informasi file atau folder.

### Fungsi `kenz_readdir`
```
static int kenz_readdir(...)
```
Fungsi ini dipanggil saat user menjalankan:
```
ls mnt
```
Fungsinya menampilkan isi folder mount.

### Fungsi `kenz_open`
```
static int kenz_open(const char *path, struct fuse_file_info *fi)
```
Fungsi ini dipanggil saat file dibuka.

Contoh:
```
cat mnt/1.txt
cat mnt/tujuan.txt
```

### Fungsi `kenz_read`
```
static int kenz_read(...)
```
Fungsi ini dipanggil saat isi file dibaca.

Contoh:
```
cat mnt/1.txt
cat mnt/tujuan.txt
```

### Struktur operasi FUSE
```
static struct fuse_operations kenz_oper = {
    .getattr = kenz_getattr,
    .readdir = kenz_readdir,
    .open = kenz_open,
    .read = kenz_read,
};
```
Ini seperti daftar fungsi yang dipakai FUSE.

### Fungsi `main`
```
int main(int argc, char *argv[])
```
Program mulai dari sini.

```
if (argc < 3) {
    fprintf(stderr, "Usage: %s <source_dir> <mount_dir>\n", argv[0]);
    return 1;
}
```
Program harus dijalankan dengan 2 argumen:
```
./kenz_rescue <folder_asli> <folder_mount>
```
Contoh:
```
./kenz_rescue amba_files mnt
```

```
realpath(argv[1], source_dir);
```
Mengubah path folder asli menjadi path lengkap.

```
char *fuse_argv[2];
fuse_argv[0] = argv[0];
fuse_argv[1] = argv[2];
```
Menyiapkan argumen untuk FUSE.

```
umask(0);
```
Agar permission file tidak otomatis dibatasi oleh sistem.

```
return fuse_main(fuse_argc, fuse_argv, &kenz_oper, NULL);
```
Menjalankan filesystem FUSE dengan operasi yang sudah dibuat.
## Output

### Ambil `amba_files.zip` dari `Flashdisk`
```
mv ~/Downloads/amba_files.zip ~/SISOP-4-2026-IT-013/soal_1/
```
Output

![App Screenshot](https://github.com/nadyaaee/SISOP-4-2026-IT-013/blob/main/assets/zip.png?raw=true?raw=true)

### Unzip file
```
unzip amba_files.zip
```
Output

![App Screenshot](https://github.com/nadyaaee/SISOP-4-2026-IT-013/blob/main/assets/unzip%20amba-files.png?raw=true)

### Hapus file zip
```
rm amba_files.zip
```
Output

![App Screenshot](https://github.com/nadyaaee/SISOP-4-2026-IT-013/blob/main/assets/hapus%20zip.png?raw=true)

### Install FUSE
```
sudo apt update
sudo apt install libfuse-dev fuse -y
```

### Compile
```
gcc -Wall `pkg-config fuse --cflags` kenz_rescue.c -o kenz_rescue `pkg-config fuse --libs`
```
Output

![App Screenshot](https://github.com/nadyaaee/SISOP-4-2026-IT-013/blob/main/assets/compile.png?raw=true)


### Buat Mount Directory
```
mkdir mnt
```
Run
```
./kenz_rescue amba_files mnt
```
Output

![App Screenshot](https://github.com/nadyaaee/SISOP-4-2026-IT-013/blob/main/assets/folder%20mnt.png?raw=true)

### Testing
Cek bahwa file virtual tidak ada di source
```
ls mnt/
```
```
ls amba_files/
```
Output 

![App Screenshot](https://github.com/nadyaaee/SISOP-4-2026-IT-013/blob/main/assets/isi%20mnt%20dan%20amba_files.png?raw=true)


Cek file passthrough
```
cat mnt/1.txt
```
Harus sama dengan:
```
cat amba_files/1.txt
```
Output

![App Screenshot](https://github.com/nadyaaee/SISOP-4-2026-IT-013/blob/main/assets/isi%20txt%201.png?raw=true)

### Validasi Semua File
```
for i in 1 2 3 4 5 6 7; do
    diff mnt/$i.txt amba_files/$i.txt && echo "$i.txt OK"
done
```
Output

![App Screenshot](https://github.com/nadyaaee/SISOP-4-2026-IT-013/blob/main/assets/ok.png?raw=true)

### Cek file virtual
```
cat mnt/tujuan.txt
```
Output

![App Screenshot](https://github.com/nadyaaee/SISOP-4-2026-IT-013/blob/main/assets/tujuan%20txt.png?raw=true)


### Unmount
```
fusermount -u mnt
```
atau
```
sudo umount mnt
```



