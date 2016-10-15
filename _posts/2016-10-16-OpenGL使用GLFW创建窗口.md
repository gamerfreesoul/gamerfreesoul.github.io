---
layout: post
title: opengl之GLFW的使用
categories: opengl
---
# OpenGL使用GLFW创建窗口

  [GLFW](http://www.glfw.org)是支持Opengl和Opengl ES，用来管理窗口，读取输入，处理事件等的libaray用来取代以前使用的glut。GLFW是一个轻量级，开源并且跨平台的lib.
###使用GLFW
  先[这里](www.glfw.ort/download.html)下载编译好的lib。使用vs创建一个工程，并添加对应的头文件和对应vs版本的库文件目录并添加lib。`如果使用动态库那么则添加glfw3dll.lib，使用静态库则添加glfw3.lib`。
  环境配置好以后就可以添加对应的代码渲染窗口：
  ---
  	#include <stdio.h>	#include "GLFW/glfw3.h"	static void GLError_CallBack(int nErrCode, const char* desc)	{		printf("Error %d: %s\n", nErrCode, desc);	}	static void GLMouseButtonClick(GLFWwindow* window, int nButton, int nAction, int mods)	{		//当左键点击时 设置关闭标记		if (nButton == GLFW_MOUSE_BUTTON_LEFT)			glfwSetWindowShouldClose(window, GLFW_TRUE);	}	int main(int arg, char** args)	{		//设置错误回调函数		glfwSetErrorCallback(GLError_CallBack);		if (!glfwInit())			return -1;		//创建窗口		GLFWwindow* window = glfwCreateWindow(1280, 720, "Test SmartUI Window", NULL, NULL);		if (!window)		{			glfwTerminate();			return -1;		}		glfwMakeContextCurrent(window);		//设置鼠标左键回调		glfwSetMouseButtonCallback(window, GLMouseButtonClick);		//循环		while (!glfwWindowShouldClose(window))		{			//渲染			glClear(GL_COLOR_BUFFER_BIT);			glBegin(GL_TRIANGLES);			glColor3f(1.0, 0.0, 0.0);			glVertex3f(0.0, 1.0, 0.0);			glColor3f(0.0, 1.0, 0.0);			glVertex3f(-1.0, -1.0, 0.0);			glColor3f(0.0, 0.0, 1.0);			glVertex3f(1.0, -1.0, 0.0);			glEnd();			//Swap front and back buffers			glfwSwapBuffers(window);			//Poll for and process events			glfwPollEvents();		}		glfwTerminate();		return 0;	}
	
  渲染一个三角形。