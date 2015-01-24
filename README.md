
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>
#include <string.h>
#include <unistd.h>
#include <netdb.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <sys/types.h>
#include <arpa/inet.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <time.h>
#define PACK_SIZE 2048*2048

char* get_file_name(char* fn);
unsigned long get_file_size(const char *path);

int main(int argc, char *argv[])
{
	time_t t_start,t_end;
	if(argc < 2)
	{
		printf("please input filePath\n");
		return 0;
	}

	// 设置输出缓冲
	setvbuf(stdout, NULL, _IONBF, 0);
	fflush(stdout);

	char* filePath = argv[1];
	if(access(filePath, F_OK) != 0)
	{
		printf("file not existed!\n");
		return 0;
	}

	int sockfd;
	char buff[1024] = {'\0'};
	struct sockaddr_in server_addr;
	struct hostent *host;
	int portnumber,nbytes;

	const char* ip = "127.0.0.1";
	host=gethostbyname(ip);
	const char* port = "8080";
	portnumber=atoi(port);

	/* 客户程序开始建立 sockfd描述符  */
	if((sockfd=socket(AF_INET,SOCK_STREAM,0))==-1)
	{
		printf("socket error\n");
		exit(1);
	}

	/* 客户程序填充服务端的资料       */
	bzero(&server_addr,sizeof(server_addr));
	server_addr.sin_family=AF_INET;
	server_addr.sin_port=htons(portnumber);
	server_addr.sin_addr=*((struct in_addr *)host->h_addr);

	/* 客户程序发起连接请求         */
	if(connect(sockfd,(struct sockaddr *)(&server_addr),sizeof(struct sockaddr))==-1)
	{
		printf("connect error\n");
		exit(1);
	}

	/* 连接成功             */
	if((nbytes=read(sockfd,buff,1024))==-1)
	{
		printf("read error\n");
		exit(1);
	}
	buff[nbytes]='\0';
	printf("I have received:%s\n",buff);

	/******* 发送指令 ********/
	bzero(buff,1024);
	// 指令ID
	int order = 0x0000;
	int order_h = order >> 8;
	buff[0] = (char)order_h;
	buff[1] = (char)order;

	// 文件长度
	unsigned long len = get_file_size(filePath);
	printf("file size = %lu\n", len);

	// 高16位
	int len_h = len >> 16;
	int len_h_1 = len_h >> 8;
	buff[2] = (char)len_h_1;
	buff[3] = (char)len_h;

	// 低16位
	int len_l = len;
	int len_l_1 = len_l >> 8;
	buff[4] = (char)len_l_1;
	buff[5] = (char)len_l;

	// 文件名称
	char* fileName = get_file_name(filePath);
	printf("file name = %s\n", fileName);
	strncpy(&buff[6], fileName, strlen(fileName));

	write(sockfd,buff,1024); 

	//计时开始
	t_start=time(NULL);

	/******* 发送文件 ********/
	printf("file path = %s\n", filePath);
	FILE* pf = fopen(filePath, "rb");
	if(pf == NULL)
	{
		printf("open file error\n");
		exit(0);
	}

	char pack[PACK_SIZE] = {'\0'};
	while((len = fread(pack, sizeof(char), PACK_SIZE, pf)) > 0)
	{
		write(sockfd, pack, len);
		bzero(pack,PACK_SIZE);
	}

	//计时结束
	t_end=time(NULL);
	printf("%fs\n",difftime(t_end,t_start));

	/* 结束通讯     */
	close(sockfd);
	exit(0);
}

char* get_file_name(char* fn)
{
	int last = 0;
	char* pfn = fn+strlen(fn)-1;
	int i=0;
	for(i=0; i<strlen(fn); ++i)
	{
		if(*pfn-- == '/')
		{
			last = strlen(fn)-i;
			break;
		}
	}
	char* name = (char*)malloc(sizeof(char)*256);
	char* pname = name;
	int j=0;
	for(j=last; j<strlen(fn); ++j, ++pname)
		*pname = fn[j];
	return name;
}

unsigned long get_file_size(const char *path)
{
	unsigned int filesize = 0;
	struct stat statbuff;
	if(stat(path, &statbuff) < 0)
	{
		printf("Get file stat failed!\n");
		return filesize;
	}
	else
		filesize = statbuff.st_size;
	return filesize;
}
