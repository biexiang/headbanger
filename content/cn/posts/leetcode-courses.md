---
title: "Leetcode Courses Problem"
date: 2022-02-13T18:45:24+08:00
draft: false
categories: ['Algorithm','Leetcode']
keywords: ['Golang','LeetCode','207.课程表','210.课程表二','630.课程表3','算法']
description: "Leetcode上课程相关的题目，包括207.课程表、210.课程表二、630.课程表3"
---

{{< music id="1341502549" >}}


从有向无环图搜到了课程表的题目，这里记录下这三个题目。测试代码：[courses_test.go](https://github.com/biexiang/code-snippet/blob/main/leetcode/courses/courses_test.go)

## 207.课程表
使用DFS查找课程表的依赖关系。
{{< highlight go "linenos=table,linenostart=1" >}}
// state: 0 means not begin
// state: 1 means begin but not finish
// state: 2 means finish
func canFinishDFS(numCourses int, prerequisites [][]int) bool {
	var (
		stateRec map[int]int
		edgeRec  map[int][]int
		dfs      func(courseID int) bool
	)
	stateRec, edgeRec = make(map[int]int, numCourses), make(map[int][]int)
	for _, item := range prerequisites {
		if edgeRec[item[1]] == nil {
			edgeRec[item[1]] = append([]int{}, item[0])
		} else {
			edgeRec[item[1]] = append(edgeRec[item[1]], item[0])
		}
	}

	dfs = func(courseID int) bool {
		stateRec[courseID] = 1
		for _, nextCourseID := range edgeRec[courseID] {
			if stateRec[nextCourseID] == 0 {
				if dfs(nextCourseID) == false {
					return false
				}
			}
			if stateRec[nextCourseID] == 1 {
				return false
			}
		}
		stateRec[courseID] = 2
		return true
	}

	for i := 0; i < numCourses; i++ {
		if stateRec[i] == 0 {
			if dfs(i) == false {
				return false
			}
		}
	}
	return true
}
{{< / highlight >}}


## 210.课程表二
使用BFS查找课程依赖。

{{< highlight go "linenos=table,linenostart=1" >}}

func findOrder(numCourses int, prerequisites [][]int) []int {
	var (
		edgeRec        map[int][]int
		verticesWeight map[int]int
		cleanCourses   []int
		result         []int
	)
	edgeRec, verticesWeight = make(map[int][]int, numCourses), make(map[int]int, numCourses)
	for _, item := range prerequisites {
		// 前置课程对应的其他课程
		if edgeRec[item[1]] == nil {
			edgeRec[item[1]] = append([]int{}, item[0])
		} else {
			edgeRec[item[1]] = append(edgeRec[item[1]], item[0])
		}
		// 若课程有其他前置课程依赖，则加一
		verticesWeight[item[0]]++
	}

	for i := 0; i < numCourses; i++ {
		if verticesWeight[i] == 0 {
			cleanCourses = append(cleanCourses, i)
		}
	}

	// 因为第一门课程总是没有前置依赖的
	for len(cleanCourses) > 0 {
		courseID := cleanCourses[0]
		cleanCourses = cleanCourses[1:]
		result = append(result, courseID)
		for _, otherCourseID := range edgeRec[courseID] {
			if verticesWeight[otherCourseID] > 0 {
				verticesWeight[otherCourseID]--
			}
			if verticesWeight[otherCourseID] == 0 {
				cleanCourses = append(cleanCourses, otherCourseID)
			}
		}
	}

	if len(result) != numCourses {
		return []int{}
	}
	return result
}
{{< / highlight >}}


## 630.课程表三
贪心计算最多可以上多少课，证明过程参考leetcode上的题解，sort和heap的源码可以参考 [golang-sort]({{<ref "/posts/golang-sort">}})。

{{< highlight go "linenos=table,linenostart=1" >}}
package courses

import (
	"container/heap"
	"sort"
)

type Heap struct {
	sort.IntSlice
}

func (h *Heap) Less(i, j int) bool {
	return h.IntSlice[i] > h.IntSlice[j]
}

func (h *Heap) Push(x interface{}) {
	h.IntSlice = append(h.IntSlice, x.(int))
}

func (h *Heap) Pop() interface{} {
	tmp := h.IntSlice
	last := tmp[len(tmp)-1]
	h.IntSlice = tmp[:len(tmp)-1]
	return last
}

// https://leetcode-cn.com/problems/course-schedule-iii/
// Greedy Algorithm
func scheduleCourse(courses [][]int) int {
	// 按照deadline排个序
	sort.Slice(courses, func(i, j int) bool {
		return courses[i][1] < courses[j][1]
	})

	h := &Heap{}
	total := 0
	// 遍历有序courses，不满足deadline的course和用时最长的比较，如果比较长，则替换
	for _, course := range courses {
		if total+course[0] <= course[1] {
			total += course[0]
			heap.Push(h, course[0])
		} else if h.Len() > 0 && course[0] < h.IntSlice[0] {
			total = total - h.IntSlice[0] + course[0]
			h.IntSlice[0] = course[0]
			heap.Fix(h, 0)
		}
	}
	return h.Len()
}
{{< / highlight >}}
