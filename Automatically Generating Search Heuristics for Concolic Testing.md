# Automatically Generating Search Heuristics for Concolic Testing

## 1. Introduction
* concolic testing은 기본적으로 구체적인 프로그램 실행과 함께 기호적으로 프로그램을 실행하면서 path condition을 모은다.
* 프로그램은 random input을 맨 처음에 입력받아 실행한 후 현재 path의 branch 들 중에 하나를 골라 그것을 negate 해서 이전에 탐색하지 않은 path를 다음 실행에 탐색할 수 있도록 새 input을 찾는다.
* conolic testing에서 핵심적인 부분은 search heuristic이다. path-explosion 문제로 인해 모든 실행 경로를 탐색하는것은 불가능하다 -> 따라서 제한된 time budget 안에 code coverage 를 최대화 하기 위해 search heuristic에 기댈 수 밖에 없다.
  * search heuristic은 기준을 가지고 negate할 최고의 branch를 찾으면서 concolic testing을 진행시킨다.
      * CFDS(Control-Flow Directed Search) : 아직 탐색되지 않은 region에서 가장 가까운 branch를 선택한다.
      * CGS(Context-Guided Search) : 새 context 안에 있을 경우에만 branch를 선택한다. (gpt: CGS는 실행빈도, 프로그램 구조 분석, 과거 실행 정보, 경로다양성, 코드 변경 이력 등의 기준을 이용하여 버그가 일어날 가능성이 큰 path를 고르도록 하는 것 같다.)
   
[문제의식]   
heuristic을 수동으로 디자인 하는 것은 힘든 일이며 불안정한 결과를 낼 수 있다. (실질적으로 현재 수동으로 디자인 된 heuristic들은 code coverage 면에서 부진함을 보이고 있다.) 뿐만 아니라 많은 engineering effort와 도메인 전문지식을 요구하기도 한다.   

* 본 논문에서는 위와 같은 문제를 해결하기 위해 concolic testing에 사용되는 search heuristic을 자동으로 생성하는 새로운 접근을 제시하고자 한다.
* 1) parameterized search heuristic을 정의하고 이를 이용해 좋은 search heuristic을 만들어 내는 문제를 좋은 parameter value를 찾는 문제로 환원시킨다.
  2) concolic testing에 특화된 search algorithm을 제시한다. parameterized search heuristic 이 제시하는 search heuristic은 너무 광범위하기 때문에 이전 concolic testing의 실행으로부터 얻은 feedback을 가지고 search space를 반복적으로 개선해서 효과적으로 탐색을 실행할 수 있게 한다.

## 2. Preliminaries
### 2.1 Concolic Testing
program P   
initial input : $v_{0}$  
symbolic memory : program variables -> symbolic values
symbolic memory state : S  
path condition : $\Phi$

* 첫 실행 후 path condition을 $\Phi = \phi_{1} \wedge \phi_{2} \wedge \dots \wedge \phi_{n}$ 라 표현하자.
* concolic testing은 branch condition $\phi_{i}$를 하나 고르고, 새로운 path condition인 $\Phi'$를 생성한다.
* $\Phi' = \bigwedge_{j < i} \phi_{j} \wedge \lnot\phi_{i}$
* 새 condition인 $\Phi'$는 기존의 $\Phi$를 i 번째 branch 이전까지 그대로 두고 i번째 branch만 negate한 것이다. $\Phi'$ 조건을 만족시키는 input value는 프로그램으로 하여금 $\phi_{i}$와 반대되는 path를 탐색하도록 할 것이다.


(algorith 1 참고)
* N : testing budget (i.e. the number of executions of the program)
* 알고리즘은 execution tree T(이전에 탐색된 path conditions들의 리스트)를 유지한다.
* line 4의 $\Phi_{m}$은 이번 실행으로 탐색한 path를 의미한다.
* line 7-9에서는 Choose 함수가 T에서 path condition $\Phi$를 고르고 그 안에서 negate할 branch인 $\phi_{i}$를 고른다. 그러면 새로운 path condition인 $\Phi' = \bigwedge_{j < i} \phi_{j} \wedge \lnot\phi_{i}$ 를 만들 수 있다. SAT 함수는 만들어진 $\Phi'$가 만족 가능한 조건인지 검사한다. 만약 만족 가능한 조건이라면 model 함수가 $\Phi$를 만족하는 input v를 생성한다.

### 2.2 Existing Search Heuristics
* Control-Flow Directed Search (CFDS) : 현재 실행 path에서 가까운 아직 실행되지 않은 branch들이 다음 실행에서 탐색될 가능성이 높다고 판단하는 가정 하에 search를 진행한다. 마지막 path condition인 $\Phi_{m}$을 선택한다음 그 중 opposite branch가 아직 탐색되지 않은 branch들과 가장 가까운 branch를 고른다. 두 branch간의 거리는 출발지부터 목적지까지 가는 path에 있는 branch의 개수이다. 거리 계산을 위해 CFDS는 프로그램의 control flow graph를 사용한다.
* Context-Guided Search (CGS) : execution tree에서 BFS를 사용해 search를 진행한다. 이 과정에서 'context'가 이미 탐색되었다면 해당 branch를 제외하는 식으로 search space를 줄여나간다. 어떤 실행 path가 있을 때, branch의 context는 해당 branch보다 앞선 branch들의 배열을 말한다. search를 진행할 때 execution tree에서 깊이 d에 있는 후보 branch들을 모으고, 그 중 한 branch를 골라 context를 계산한다. 만약 context가 이미 고려되었다면, 다른 branch를 선택하고 그 branch를 negate한 뒤에 context가 저장된다. 깊이 d에 있는 모든 후보 branch 들이 모두 탐색되었다면 search는 깊이가 d+1인 곳으로 가서 다시 앞의 과정을 반복한다.

* 한계점 : 기존의 search heuristics는 고정된 heuristic을 사용하여 다양한 범위의 target program에서 일관되게 잘 작동하지 못한다. 그리고 좋은 search heuristic을 수동으로 만드는 데에는 너무 많은 노력과 전문성이 필요하다.

 ## 3. Our Technique
 ### 3.1 Paraeterized Search Heuristic
 *  Choose $\in$ SearchHeuristic = ExecutionTree $\rightarrow$ PathCond $\times$ Branch
 *  family $H \subseteq$ SearchHeuristic of search heuristics를 parameterized heuristic $Choose_{\theta}$라 하자.
 *  $\theta$는 k-차원 실수 벡터인 parameter이다.
 *  $H = \\{ Choose_{\theta} \mid \theta \in \mathbb{R}^{k} \\}$
 *  execution tree T = $<\Phi_{1}\Phi_{2}\cdots\Phi_{m}>$
 *  $Choose_{\theta}(<\Phi_{1} \cdots \Phi_{m}>) = (\Phi_{m},argmax_{\phi_{j} \in \Phi_{m}} score_{\theta}(\phi_{j}))$
 *  해당 heuristic은 execution tree T에서 마지막 path condition인 $\Phi_{m}$을 선택하고, $\Phi_{m}$ 중에서 가장 높은 점수를 가진 branch $\phi_{j}$ 를 선택한다.
 *  $\Phi_{m}$ 안의 각 branch인 $\phi$의 점수가 parameter인 $\theta$에 따라 어떻게 매겨지는지는 다음과 같다.
    1) feature vector로 각 branch를 표현한다. feature인 $\pi_{i}$는 branch에 대한 boolean 서술값이다. $\pi_{i}: Branch $\rightarrow {0,1}$ 예를 들어, 어떤 feature는 해당 branch가 main function에 위치하는지 아닌지를 판단한다. k개의 feature들이 있다 하면, $\pi = {\pi_{1}, \cdots, \pi_{k}}$ 이다. k가 parameter $\theta$의 길이라 하면, branch $\phi$ 는 다음과 같은 boolean vector로 표현된다. $$\pi(\phi) = <\pi_{1}(\phi),\pi_{2}(\phi),\cdots,\pi_{k}(\phi)>$$
    2) branch의 점수를 구한다. parameter $\theta$의 k 차원은 branch feature의 개수와 같다. $$ score_{\theta}(\phi) = \pi(\phi)\cdot\theta $$
    3) 가장 높은 score을 가지고있는 branch를 구한다. 즉, $\Phi_{m}$ 안의 branch $\phi_{1},\cdots,\phi_{n}$ 중에서, 모든 k에 대해 $score_{\theta}(\phi_{j}) \geq score_{\theta}(\phi_{k})$ 인 branch $\phi_{j}$를 선택하는 것이다.     
       ![ex](./paper_review/images/Automatically Generating Search Heuristics for Concolic Testing/ex1.png)
    
 * Branch Features : 본 논문에서는 40개의 feature들을 디자인해놓았다. 12개의 static feature들은 프로그램을 실행하지 않고도 얻을 수 있는 property이며 나머지 28개의 dynamic feature들은 프로그램을 실행해서 그 도중에 값을 얻을 수 있는 property이다.
