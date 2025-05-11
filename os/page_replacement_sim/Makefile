CC = gcc
CFLAGS = -Wall -Wextra -g -Iinclude -std=c11 # -g用于调试
LDFLAGS =

# 查找src目录中的所有.c文件
SRCS = $(wildcard src/*.c)
# 从源文件名生成目标文件名
OBJS = $(patsubst src/%.c,obj/%.o,$(SRCS))

TARGET = page_sim

# 默认目标
all: $(TARGET)

# 链接目标可执行文件
$(TARGET): $(OBJS)
	$(CC) $(CFLAGS) $^ -o $@ $(LDFLAGS)

# 将源文件编译为目标文件
obj/%.o: src/%.c | obj
	$(CC) $(CFLAGS) -c $< -o $@

# 目标文件目录
obj:
	mkdir -p obj

# 结果目录
results:
	mkdir -p results

# 数据目录（如果保存工作负载）
data:
	mkdir -p data

# 清理构建文件
clean:
	rm -rf $(TARGET) obj/* results/* data/*

.PHONY: all clean results data