# 使用OR Tools进行线性优化

总结尝试初步使用OR Tools。

## 极简介绍

OR Tools又叫做 Google Optimization Tools，是谷歌开源的，用来解决组合优化问题的快速、可移植的软件工具包。
这个工具包提供以下几种求解器：
1. 带有约束的线性规划求解器；
2. 普通 的线性规划求解器；
3. 商业或者其他开源的求解器，包括混合整数求解器；
4. Bin打包和背包算法求解器；
5. 旅行商算法和车辆路径规划算法求解器；
6. 图类的算法求解器（包括最短路径，最小代价，最大路径等）

OR Tools提供了C++的接口，也提供了C#和Python的接口。

## 安装

支持跨平台在Windows和Linux，以及Mac OS上都支持，但是有版本和工具集的最低要求。

- Ubuntu： 16.04以上，且是64位系统
- Mac OSX El Capitan with Xcode 7.x (64-位)
- Windows: 必须有Visual Studio 2015或者2017

目前只支持用Makefile来构建，未来支持Bazel。

手上有Windows 10 和Linux 16.04两种平台，可以逐一尝试一下从源码安装，不过在文档中，建议使用预编译的二进制文件安装。

在Windows上安装需要Visual Studio 2015或者Visual Studio 2017 或者Visual Studio Community 2017.

Google很多开源工具都有详细的文档，学习起来容易的多。
在[https://developers.google.com/optimization/install/](https://developers.google.com/optimization/install/)描述了详细的安装说明，包括从源码安装和从二进制文件安装。

## 从一个简单的C++例子入手
个人更熟悉使用C++，先从C++的一个最简单的例子入手。

**问题： **解决一个受限线性规划问题，计算在一个固定区间内，x+y的最大取值，其中x的区间是[0,1], y的区间是[0,2]。

```
#include "ortools/linear_solver/linear_solver.h"
#include "ortools/linear_solver/linear_solver.pb.h"

namespace operations_research {
  void RunTest(
    MPSolver::OptimizationProblemType optimization_problem_type) {
    MPSolver solver("Glop", optimization_problem_type);
    MPVariable* const x = solver.MakeNumVar(0.0, 1, "x");
    MPVariable* const y = solver.MakeNumVar(0.0, 2, "y");
    MPObjective* const objective = solver.MutableObjective();
    objective->SetCoefficient(x, 1);
    objective->SetCoefficient(y, 1);
    objective->SetMaximization();
    solver.Solve();
    printf("\nSolution:");
    printf("\nx = %.1f", x->solution_value());
    printf("\ny = %.1f", y->solution_value());
  }

  void RunExample() {
    RunTest(MPSolver::GLOP_LINEAR_PROGRAMMING);
  }
}

int main(int argc, char** argv) {
  operations_research::RunExample();
  return 0;
}
```

1. 上面是实例代码，在安装OR-Tools的根目录中创建一个cpp文件来保存这一段代码，比如，命名为simple_program.cc。
2. 然后，同样在OR-Tools的根目录中创建一个Makefile.user文件，然后添加编译规则
```
$(OBJ_DIR)$Ssimple_program.$O: $(EX)
	$(CCC) $(CFLAGS) -c $(EX) $(OBJ_OUT)$(OBJ_DIR)$S$(basename $(notdir $(EX))).$O

$(BIN_DIR)/simple_program$E: $(OR_TOOLS_LIBS) $(OBJ_DIR)$Ssimple_program.$O
	$(CCC) $(CFLAGS) $(OBJ_DIR)$Ssimple_program.$O $(OR_TOOLS_LNK) \
 $(OR_TOOLS_LD_FLAGS) $(EXE_OUT)$(BIN_DIR)$Ssimple_program$E
```
3. 然后，打开命令行，编译并执行
```
make rcc EX=simple_program.cc
```

上面的例子输出如下
```
Solution:
x =  1.0
y =  2.0
```

## OR-Tools解决什么优化问题

优化问题是从很多的解决方案中，找出一个最优的解决方案，优化问题定义很广泛，OR-Tools是一种求解这种优化问题的一种工具。

比如，卡车送货问题，需要根据货物的重量以及路程代价，选择一种代价最小的路线和方案。

与其他的优化问题一样，问题被定义成包含目标函数和约束的形式。
- 目标函数，是指一种衡量指标，比如代价最小或者收益最大；
- 约束，是指求解的解决方案必须满足的条件。

使用工具求解优化问题，首先要做的是定义目标函数和约束。

下面用一个例子说明，如何使用OR-Tools解决优化问题。

**一个线性优化问题： **
定义线性优化的目标函数是3x + 4y，求解该目标函数的最大值，约束项有三个：
1. x + 2y	≤	14
2. 3x – y	≥	0
3. x – y	≤	2

实际上，这个问题是在一个定义的范围内，求解最大值，如下图所示：

![问题](https://developers.google.com/optimization/images/lp/feasible_region.png)

使用OR-Tools求解问题的主要步骤如下：
1. 创建变量；
2. 定义约束；
3. 定义目标函数；
4. 定义求解器，求解器是工具提供的函数；
5. 触发求解器，并且输出结果。

使用C++接口实现这样一个优化求解问题。
1. 使用`MakeNumVar`创建变量；
```
  MPVariable* const x = solver.MakeNumVar(0.0, infinity, "x");
  MPVariable* const y = solver.MakeNumVar(0.0, infinity, "y");
```

2. 使用`MakeRowConstraint `和`SetCoefficient`定义约束
```
// x + 2y <= 14.
  MPConstraint* const c0 = solver.MakeRowConstraint(-infinity, 14.0);
  c0->SetCoefficient(x, 1);
  c0->SetCoefficient(y, 2);

  // 3x - y >= 0.
  MPConstraint* const c1 = solver.MakeRowConstraint(0.0, infinity);
  c1->SetCoefficient(x, 3);
  c1->SetCoefficient(y, -1);

  // x - y <= 2.
  MPConstraint* const c2 = solver.MakeRowConstraint(-infinity, 2.0);
  c2->SetCoefficient(x, 1);
  c2->SetCoefficient(y, -1);
```

3. 定义目标函数
```
 // Objective function: 3x + 4y.
  MPObjective* const objective = solver.MutableObjective();
  objective->SetCoefficient(x, 3);
  objective->SetCoefficient(y, 4);
  objective->SetMaximization();
```

4. 声明求解器
在定义求解函数的时候，参数是求解器优化函数类型
```
void RunLinearExample(MPSolver::OptimizationProblemType optimization_problem_type)
  {

  }
```

一般例子中是在求解函数中，用传入的优化问题的类型声明一个求解器，从例子中可以看到，在这个函数体第一行就声明一个求解器。
```
MPSolver solver("LinearExample", optimization_problem_type);
```

在主函数中，调用这个优化函数，并传入想要使用的优化算法类型，比如下面是使用GLOP来优化线性规划问题。
```
RunLinearExample(MPSolver::GLOP_LINEAR_PROGRAMMING);
```

5. 使用求解器，并输出结果
```
printf("\nNumber of variables = %d", solver.NumVariables());
  printf("\nNumber of constraints = %d", solver.NumConstraints());
  solver.Solve();
  // The value of each variable in the solution.
  printf("\nSolution:");
  printf("\nx = %.1f", x->solution_value());
  printf("\ny = %.1f", y->solution_value());

  // The objective value of the solution.
  printf("\nOptimal objective value = %.1f", objective->Value());
  printf("\n");
```

完整的代码如下
```
#include "ortools/linear_solver/linear_solver.h"
#include "ortools/linear_solver/linear_solver.pb.h"

namespace operations_research {
void RunLinearExample(
  MPSolver::OptimizationProblemType optimization_problem_type) {
  MPSolver solver("LinearExample", optimization_problem_type);
  const double infinity = solver.infinity();
  // x and y are non-negative variables.
  MPVariable* const x = solver.MakeNumVar(0.0, infinity, "x");
  MPVariable* const y = solver.MakeNumVar(0.0, infinity, "y");
  // Objective function: 3x + 4y.
  MPObjective* const objective = solver.MutableObjective();
  objective->SetCoefficient(x, 3);
  objective->SetCoefficient(y, 4);
  objective->SetMaximization();
  // x + 2y <= 14.
  MPConstraint* const c0 = solver.MakeRowConstraint(-infinity, 14.0);
  c0->SetCoefficient(x, 1);
  c0->SetCoefficient(y, 2);

  // 3x - y >= 0.
  MPConstraint* const c1 = solver.MakeRowConstraint(0.0, infinity);
  c1->SetCoefficient(x, 3);
  c1->SetCoefficient(y, -1);

  // x - y <= 2.
  MPConstraint* const c2 = solver.MakeRowConstraint(-infinity, 2.0);
  c2->SetCoefficient(x, 1);
  c2->SetCoefficient(y, -1);
  printf("\nNumber of variables = %d", solver.NumVariables());
  printf("\nNumber of constraints = %d", solver.NumConstraints());
  solver.Solve();
  // The value of each variable in the solution.
  printf("\nSolution:");
  printf("\nx = %.1f", x->solution_value());
  printf("\ny = %.1f", y->solution_value());

  // The objective value of the solution.
  printf("\nOptimal objective value = %.1f", objective->Value());
  printf("\n");
}

void RunExample() {
  RunLinearExample(MPSolver::GLOP_LINEAR_PROGRAMMING);
}
}  // namespace operations_research

int main(int argc, char** argv) {
  operations_research::RunExample();
  return 0;
}
```

优化输出结果如下：
```
Number of variables = 2
Number of constraints = 3
Solution:
x = 6.0
y = 4.0
Optimal objective value = 34.0
```

关于OP-Tools如何优化问题，参考源代码中的例子。

## 参考

1. [Using OR-Tools with C++](https://developers.google.com/optimization/introduction/cpp)
2. [or-tools Github Repo](https://github.com/google/or-tools)