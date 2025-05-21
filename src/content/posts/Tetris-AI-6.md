---
title: 테트리스를 깨는 AI 제작 - 6 [본격 라이브러리 구현 1]
published: 2025-05-21
description: '테트리스를 스스로 깨는 강화학습 에이전트 CUDA부터 개발하기'
tags: ["C++","AI", "CUDA" ,"ReinforceLearning"]
category: 'Project'
draft: false 
lang: 'ko'
---

## 프로젝트 개요

AI에 대해서는 예전부터 관심이 있었고 C++과 rust같은 로우레벨 프로그래밍 언어도 개인적으로 좋아합니다.

그래서 이번 프로젝트는 C++와 CUDA를 활용해 외부 라이브러리 없이 테트리스 게임을 직접 구현하고,
이 게임을 스스로 플레이하며 최고 점수를 노리는 강화학습 에이전트를 처음부터 만들어보는 과정을 기록하는 것을 목표로 할 것입니다.

로우레벨 언어와 인공지능, 그리고 GPU 프로그래밍에 관심이 많은 학부생으로서, 이미 잘 만들어진 라이브러리나 프레임워크에 의존하지 않고
처음부터 모든 것을 직접 설계하고 구현해보는 경험을 통해, 진짜로 시스템이 어떻게 돌아가는지 깊이 이해하고 싶습니다.

또한, GPU의 병렬 연산 능력을 실제로 활용해보며, 이론으로만 배웠던 개념들이 실제 코드와 하드웨어에서 어떻게 동작하는지 체험하고자 합니다.

이 프로젝트를 통해 배우고자 하는 가장 큰 목표는, 강화학습의 핵심 원리와 GPU 프로그래밍의 실전 기술을 내 손으로 직접 구현하며 익히는 것입니다.

테트리스라는 익숙한 게임을 스스로 만들고, 그 위에서 동작하는 에이전트를 설계하면서, 상태 공간과 행동 집합, 보상 함수 설계의 중요성을 느낄 것입니다.

또한, CUDA를 활용해 대량의 시뮬레이션을 병렬로 처리하는 과정에서, GPU 메모리 관리나 커널 최적화와 같은 실전적인 문제들을 직접
해결해보고자 합니다.

라이브러리 없이 신경망이나 알고리즘을 처음부터 구현하는 과정에서, 평소에는 잘 느끼지 못했던 컴퓨팅 자원의 한계나, 병렬 연산의 어려움도
경험할 수 있을 것이라 기대하고 있습니다.

어려움도 많겠지만 이런 난관들을 직접 부딪히고 해결해가는 과정에서, 단순히 결과만 얻는 것이 아니라, 문제를 분석하고 해결책을 찾아가는 과정
자체가 큰 성장의 기회가 될 것이라고 생각합니다.

## 내용

### Tensor 클래스 구현

텐서는 신경망에서 모든 연산의 바탕으로 모든 연산은 텐서를 중심으로 이루어집니다.

제가 경험하길 Tensorflow와 Pytorch를 써본 결과 텐서라는 데이터를 모델에 흐르게 하므로써 작동했으므로 이 라이브러리에서도 텐서를 중심으로 작동하도록 설계했습니다.

텐서를 메모리에 저장할때 신경망은 성능을 최대한 끌어다 써야하기 때문에 당연히 1차원 배열형태로 저장했습니다.

하지만 텐서는 개념적으로 1차원, 2차원, 3차원, 개념적으로는 n차원도 가능하기에 `strides_`(각 차원까지 칸수)와 `shapes_`(차원) 멤버 변수를 저장하고 관련 메소드를 지원함으로써 이 개념을 구현하였습니다.

텐서는 각 텐서마다 고유한 이름과 uid를 가져야 하기에 `name_`과 `uid_` 멤버 변수를 가지고 데이터 타입을 위한 변수 `data_type_`을 가집니다.

이 때 일반적으로 신경망에서 텐서를 가장 많이 쓰는 가중치는 32비트 부동소수점으로 관리되기 때문에 float형으로 기본적으로 저장되게 설계했습니다.

```cpp
    std::vector<int64_t> shape_;
    mutable std::vector<int64_t> strides_;
    cudnn_frontend::DataType_t data_type_ = cudnn_frontend::DataType_t::FLOAT;
    int64_t uid_ = tensor_uid_counter.fetch_add(1);
    std::string name_;
```

여기서 왜 `strides_`가 mutable로 선언되어있냐고 말할 수도 있습니다.

이유는 개념적으로 내부 변수가 변경되면 안될 `void calculate_strides_if_needed() const`에서 계산을 할 때 `strides_` 변수의 업데이트가 필요해서입니다.

---

신경망은 병렬처리가 매우 핵심인 특성상 GPU를 많이 사용합니다.

따라서 제 라이브러리도 CUDA와 cuDNN을 사용하는 것이고 따라서 이 텐서도 GPU에 저장될 필요가 있습니다.

그것을 위해 `unique_ptr`, 스마트 포인터와 커스텀 소멸자로 메모리를 관리했습니다.

```cpp
struct CudaDeleter
{
    void operator()(void* ptr) const
    {
        if (ptr)
        {
            CUDA_CHECK(cudaFree(ptr));
        }
    }
};

struct HostDeleter
{
    void operator()(void* ptr) const
    {
        if (ptr)
        {
            delete[] static_cast<char*>(ptr); 
        }
    }
};

std::unique_ptr<char[], HostDeleter> h_data_ptr_;  // CPU 데이터 (바이트 배열로 관리)
std::unique_ptr<void, CudaDeleter> d_data_ptr_;    // GPU 데이터
```

하지만 여기서 문제가 발생합니다.

사용자(라이브러리 사용자, 프로그램 사용자 모두 포함)가 GPU나 CPU에 있는 텐서를 서로 다른 쪽으로 보낼 때 동기화 문제입니다.

단순하게 생각하면 그냥 상대 장치에 공간을 할당하고 복사하고 기존 공간을 지우면 되는거 아니냐고 생각할 수도 있습니다.

하지만 자료들을 찾아본 결과 기존에 있는 공간을 유지해서 필요할 때 양쪽에 있는 두 공간을 동기화 시키는 방식도 있다는 걸 알았습니다.

이 방식의 장점은 데이터를 CPU와 GPU 사이에 자주 이동시킬 때 효율적으로 동작하고 사용자에게 더 많은 최적화 가능성을 줄 수 있습니다.

하지만 단점은 구현이 복잡해지고 메모리가 낭비될 수 있다는 것입니다.

저는 생각해본 결과 일단 지금은 이동하는 방식을 택하고 나중에 좀 더 많은 지식을 습득하고 이 쪽 분야에 더 익숙해지면 이런 동기화 방식을 적용해보는 것으로 선택했습니다.

---

텐서의 device, 그러니까 GPU쪽은 안의 데이터가 GPU 내에서만 수정되고 cuda 관련 함수들은 모두 void 포인터를 인자로 받기 때문에 void 스마트 포인터로 멤버 변수를 가지게 했습니다.

하지만 host, CPU쪽은 데이터가 수정될 가능성이 조금이라도 있고 void 포인터로 받는 함수나 그런것도 없습니다.

따라서 CPU 쪽 스마트 포인터는 char형 배열의 스마트 포인터로 관리해 바이트 단위의 배열로 기본적으로 저장을 하고 `data_type_` 멤버 변수에 따라 런타임에 타입을 바꿔주어 수정할 수 있도록 설계했습니다.

### Layer 클래스 구현

레이어는 Pytorch로 치면 Linear, conv2d와 같은 신경망의 층, 말 그대로 레이어입니다.

이 레이어는 레이어 클래스를 가장 최상위 부모로 두고 그걸 상속해서 아래에 다른 실제 레이어들을 두는 것으로 설계했습니다.

레이어 클래스에는 모든 레이어에 공통적으로 들어갈 이름, `name_` 멤버 변수와 그에 관련된 메소드들이 있습니다.

그리고 신경망에서 모든 레이어들이 정의해야할 순전파, 역전파 관련 메소드들을 상속한 클래스에서 오버라이드해서 사용할 수 있도록 순수 가상 함수로 선언해뒀습니다.

```cpp
    // 순전파 그래프 구성 순수 가상 함수
    virtual std::shared_ptr<fe::graph::Tensor_attributes> add_forward_to_graph(
        std::shared_ptr<fe::graph::Graph>& graph,
        const std::shared_ptr<fe::graph::Tensor_attributes>& input_tensor_graph_ref) = 0;

    // 역전파 그래프 구성 순수 가상 함수
    virtual std::shared_ptr<fe::graph::Tensor_attributes> add_backward_to_graph(
        std::shared_ptr<fe::graph::Graph>& graph,
        const std::shared_ptr<fe::graph::Tensor_attributes>& output_grad_graph_ref,
        const std::shared_ptr<fe::graph::Tensor_attributes>& fwd_input_graph_ref,  // 순전파 시의 입력
        const std::shared_ptr<fe::graph::Tensor_attributes>&
            fwd_output_graph_ref) = 0; 
```

---

저는 일단 Q 함수를 근사하는 신경망으로 CNN이 아닌 MLP를 사용해볼 것이기 때문에 컨볼루션 레이어는 아직 구현을 하진 않았고 Tensorflow의 명칭으론 Dense 레이어, Pytorch 명칭으로는 Linear 레이어를 구현했습니다.

DenseLayer 클래스에서는 부모, Layer 클래스의 메소드들을 오버라이드해서 구현을 하고 그에 필요한 가중치와 편향, 그에 대응하는 기울기 멤버 함수들을 가지게 하였습니다.

순전파와 역전파를 구성하는 메소드들은 모두 cuDNN의 그래프를 이용하여 구현하였습니다.

정확히는 cuDNN의 그래프 API 중 frontend API 만 사용하여 구현하였습니다.

이유는 cuDNN의 최신 버전이 9.10.0 버전이였는데 9버전 들어와서 백엔드쪽 API가 많은 API가 Deprecated되고 추가되고 삭제되는 등 뭔가 변화가 진행중인것 같아서였습니다.

다시 돌아와서 순전파와 역전파 모두 cuDNN의 graph에 추가해서 연산하는 식으로 동작하게 설계했습니다.

순전파에서는 Dense 레이어의 공식인 $XW+b=Y$에 따라 행렬 곱과 덧셈 연산을 정의하는 그래프 속성 객체를 만들고 그래프에 각 텐서를 등록하고 가상 텐서로 만들어 실행하는 구조를 가지게 하였습니다.

```cpp
std::shared_ptr<fe::graph::Tensor_attributes> DenseLayer::add_forward_to_graph(
    std::shared_ptr<fe::graph::Graph>& graph,
    const std::shared_ptr<fe::graph::Tensor_attributes>& input_tensor_graph_ref)
{
    auto matmul_operation = fe::graph::Matmul_attributes().set_compute_data_type(weights_.get_data_type());
    auto add_operation = fe::graph::Pointwise_attributes()
                             .set_compute_data_type(bias_.get_data_type())
                             .set_mode(fe::PointwiseMode_t::ADD);

    auto weights_tensor_attributes = weights_.create_graph_tensor_attributes(graph);
    auto bias_tensor_attributes = bias_.create_graph_tensor_attributes(graph);

    // 가중치 행렬곱
    auto wegihts_output_tensor_attrubutes =
        graph->tensor(graph->matmul(input_tensor_graph_ref, weights_tensor_attributes, matmul_operation)
                          ->set_is_virtual(true)
                          .set_name(name_ + "_weights_matmul_out"));

    // 편향 덧셈
    auto output_tensor_attributes =
        graph->tensor(graph->pointwise(wegihts_output_tensor_attrubutes, bias_tensor_attributes, add_operation)
                          ->set_is_virtual(true)
                          .set_name("_bias_add_out"));

    return output_tensor_attributes;
}
```

역전파에서는 전에 증명했듯이 가중치의 손실함수에 대한 미분은 다음 층에서 흘러들어온 기울기에 입력 텐서를 전치한걸 행렬 곱 해준 값을 저장해줬고 편향에 대한 미분은 앞쪽에서 하진 않았지만 흘러들어온 기울기를 배치축에 대해 합산하는 것입니다.

입력에 대한 손실함수의 미분도 앞서 한것처럼 흘러들어온 기울기의 전치와 가중치의 행렬곱이기에 행렬곱 연산 객체를 생성하고 그래프에 추가하고 구성했습니다.

```cpp
std::shared_ptr<fe::graph::Tensor_attributes> DenseLayer::add_backward_to_graph(
    std::shared_ptr<fe::graph::Graph>& graph,
    const std::shared_ptr<fe::graph::Tensor_attributes>& output_grad_graph_ref,
    const std::shared_ptr<fe::graph::Tensor_attributes>& fwd_input_tensor_ref,  // 순전파 시의 입력
    const std::shared_ptr<fe::graph::Tensor_attributes>& fwd_output_tensor_ref)

{
    auto matmul_operation = fe::graph::Matmul_attributes().set_compute_data_type(weights_grad_.get_data_type());
    auto weights_grad_tensor_attributes = weights_grad_.create_graph_tensor_attributes(graph);

    // 순전파 입력 행렬 전치
    auto fwd_input_tensor_ref_shape = fwd_input_tensor_ref->get_dim();
    auto fwd_input_tensor_ref_shape_size = fwd_input_tensor_ref_shape.size();
    std::swap(fwd_input_tensor_ref_shape[fwd_input_tensor_ref_shape_size - 1],
              fwd_input_tensor_ref_shape[fwd_input_tensor_ref_shape_size - 2]);

    auto fwd_input_graph_ref_T =
        graph->tensor(fwd_input_tensor_ref->set_dim(fwd_input_tensor_ref_shape).set_is_virtual(true));

    // 손실함수에 대한 가중치의 기울기 계산
    weights_grad_tensor_attributes =
        graph->tensor(graph->matmul(fwd_input_graph_ref_T, output_grad_graph_ref, matmul_operation)
                          ->set_is_virtual(true)
                          .set_name(name_ + "_weights_matmul_out_bwd"));

    // reduction 작업 attributes 정의
    auto reduction_operation = fe::graph::Pointwise_attributes()
                                   .set_compute_data_type(bias_grad_.get_data_type())
                                   .set_mode(fe::PointwiseMode_t::ADD)
                                   .set_axis(0);
    auto bias_grad_tensor_attributes = bias_grad_.create_graph_tensor_attributes(graph);

    // 손실함수에 대한 편향의 기울기 계산
    bias_grad_tensor_attributes = graph->tensor(graph->pointwise(output_grad_graph_ref, reduction_operation)
                                                    ->set_is_virtual(true)
                                                    .set_name(name_ + "_bias_add_out_bwd"));

    // 가중치 행렬 전치
    auto weights_tensor_attribute = weights_.create_graph_tensor_attributes(graph);
    auto weights_tensor_shape = weights_grad_tensor_attributes->get_dim();
    auto weights_tensor_shape_size = weights_tensor_shape.size();
    std::swap(weights_tensor_shape[weights_tensor_shape_size - 1], weights_tensor_shape[weights_tensor_shape_size - 2]);

    auto weights_tensor_T = graph->tensor(weights_tensor_attribute->set_dim(weights_tensor_shape).set_is_virtual(true));

    // 손실함수에 대한 입력의 기울기 계산
    return graph->tensor(graph->matmul(output_grad_graph_ref, weights_tensor_T, matmul_operation)
                             ->set_is_virtual(true)
                             .set_name(name_ + "_output_bwd"));
}
```

순전파 출력 텐서는 필요없는데 왜 받았냐고 할 수 있는데 그것은 가상 함수를 정의할 때 아직 다른 레이어는 미분을 유도해보지 않아서 잘 몰라 다른 레이어 클래스에서 사용할까봐 남겨놨습니다.

## 회고 및 앞으로 할 일

이 텐서 클래스와 레이어 클래스를 만들 때 GPU를 쉽게 사용하기 위해 cuDNN을 사용했는데 cuDNN이 개편되고 있어서 자료가 마구 섞여있는데다가 애초에 자료 자체도 많이 없어서 거의 공식 깃헙 예제와 공식 문서로만 알아내야해서 그 점이 힘들었습니다.

## 프로젝트 깃헙

::github{repo="Ushio-Hayase/Ushionn"}
