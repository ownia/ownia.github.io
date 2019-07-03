---
layout: post
title: "扫雷MineSweeper的设计"
---
#### MineSweeper的实现思路

* 使用二维的vector容器数组存储gameMap
* 使用结构体MineBlock存储gameMap信息
* 使用枚举法判断周围的地雷数量
* 使用递归法进行挖雷
* 计算周围地雷数量时要排除本身
* 标记分为错误标记(MARKED)和正确标记(WRONG_MARK)，需要对两个标记进行区分(在ui层游戏结束时做绘制区分)
* 一个方块如果标记两次会回到UNDIG的状态

在构造函数里面不做具体初始化，因为构造函数的调用时间不确定
界面设置
```c++
blockSize = 20; //方块大小
offsetX = 5; //横向边距
offsetY = 5; //纵向边距
scroeboardY = 70; //记分板高度
```
设置窗口控件固定大小 
```
game -> mCol * blockSize + offsetX * 2
game -> mRow * blockSize + offsetY * 2 + scroeboardY
(game -> mCol) * 20 + 5 * 2
(game -> mRow) * 20 + 5 * 2 + 70
```
drawPixmap()使用GPU处理，相对减轻了CPU的负担

`void QPainter::drawPixmap(int x, int y, const QPixmap & pixmap, int sx = 0, int sy = 0, int sw = -1, int sh = -1)`
通过把pixmap的一部分复制到绘制设备中，在(x, y)绘制一个像素映射。
(x, y)指定了要被绘制的绘制设备的左上点
(sx, sy)指定了要被绘制的pixmap中的左上点，默认为(0, 0)。
(sw, sh)指定了要被绘制的pixmap的大小，默认(-1, -1)，意思是一直到像素映射的右下。
当在QPrinter上绘制时，当前像素映射的遮蔽或者它的alpha通道被忽略。

游戏的四种状态
```
case OVER:
painter.drawPixmap((game -> mCol * blockSize + offsetX * 2) / 2 - 12, scroeboardY / 2 + 3.5, bmpFaces, 0 * 24, 0, 24, 24);
break;
case PLAYING:
painter.drawPixmap((game -> mCol * blockSize + offsetX * 2) / 2 - 12, scroeboardY / 2+ 3.5, bmpFaces, 1 * 24, 0, 24, 24);
break;
case WIN:
painter.drawPixmap((game -> mCol * blockSize + offsetX * 2) / 2 - 12, scroeboardY / 2+ 3.5, bmpFaces, 2 * 24, 0, 24, 24);
break;
default:
painter.drawPixmap((game -> mCol * blockSize + offsetX * 2) / 2 - 12, scroeboardY / 2+ 3.5, bmpFaces, 1 * 24, 0, 24, 24);
break;
```
`((game -> mCol) * 20 + 5 * 2))/2 -12`先取控件宽度中心，再减去图像大小24的一半12，获得x轴位置
`70 / 2 + 3.5 = 38.5`取记分板高度70的一半加3.5，获得y轴位置
`0 * 24`第一个图形的sx值
`1 * 24`第二个图形的sx值
`2 * 24`第三个图形的sx值
三个图形的sy值都为0
要绘制的pixmap大小即为24
```c++
int n = game -> currentMineNumber;
int posX = 50;
if(n < 0)
{
	painter.drawPixmap(posX, scroeboardY / 2 + 1.5, bmpTimeNumber, n * 20, 0, 20, 28);
}
while(n > 0)
{
	painter.drawPixmap(posX - 20, scroeboardY / 2 + 1.5, bmpTimeNumber, 	n % 10 * 20, 0, 20, 28); //每次从后面绘制一位
	n /= 10;
	posX -= 20;
}
```
`posX = 50`获得x轴位置
`70 / 2 + 1.5 = 36.5`取记分板高度70的一半加1.5，获得y轴位置
`n % 10 * 20`先取最后一位数字
`n /= 10, posX -= 20`向前移一位，取得第一位数字

currentState的状态
```c++
switch(game -> gameMap[i][j].currentState)
{
	case UNDIG:
		painter.drawPixmap(j * blockSize + offsetX, i * blockSize + offsetY + scroeboardY , bmpBlocks, blockSize * 10, 0, blockSize, blockSize);
		break;
	case DIGGED:
		painter.drawPixmap(j * blockSize + offsetX, i * blockSize + offsetY + scroeboardY, bmpBlocks, blockSize * game -> gameMap[i][j].valueFlag, 0, blockSize, blockSize);
		break;
	case MARKED:
		painter.drawPixmap(j * blockSize + offsetX, i * blockSize + offsetY + scroeboardY, bmpBlocks, blockSize * 11, 0, blockSize, blockSize);
		break;
	case BOMB:
		painter.drawPixmap(j * blockSize + offsetX, i * blockSize + offsetY + scroeboardY, bmpBlocks, blockSize * 9, 0, blockSize, blockSize);
		break;
	case WRONG_MARK:
		if(game->gameState == PLAYING || game->gameState == FAULT)
		{
			//如果还在游戏中就显示旗子
			painter.drawPixmap(j * blockSize + offsetX, i * blockSize + offsetY + scroeboardY, bmpBlocks, blockSize * 11, 0, blockSize, blockSize);
		}
		else if(game->gameState == OVER)
		{
			//如果游戏已经结束，就显示标错了
			painter.drawPixmap(j * blockSize + offsetX, i * blockSize + offsetY + scroeboardY, bmpBlocks, blockSize * 12, 0, blockSize, blockSize);
		}
		break;
	default:
		break;
}
```
`game -> gameMap[i][j].currentState`获得该方块的状态
`j * 20 + 5`获得x轴位置
`i * 20 + 5 + 70`获得y轴位置
根据状态UNDIG，DIGGED，MARKED，BOMB，WRONG_MARK分别绘制相应图像
其中比较特殊的是DIGGED和WRONG_MARK

DIGGED的pixmap的sx选取是通过`20 * game -> gameMap[i][j].valueFlag`来选取
WRONG_MARK的对应绘制图像是要先判断游戏是否结束，然后分别显示不同的图像
```c++
if(x >= (game -> mCol * blockSize + offsetX * 2) / 2 - 12 && x <= (game -> mCol * blockSize + offsetX * 2) / 2 + 12 && y >= scroeboardY / 2 + 3.5 && y <= scroeboardY / 2 + 3.5 + 24)
```
判断鼠标指针是否点击了faces图案
```c++
if(x >= (game -> mCol * 20 + 5 * 2) / 2 - 12 && x <= (game -> mCol * 20 + 5 * 2) / 2 + 12 && y >= 70 / 2 + 3.5 && y <= 70 / 2 + 3.5 + 24)
```
计算得出鼠标所点击的block的row值和col值
```c++
int px = event->x() - offsetX;
int py = event->y() - offsetY - scroeboardY;

int row = py / blockSize;
int col = px / blockSize;
```
