# 2만 개 노드를 가진 메뉴 트리의 최초 로딩 속도 15초 > 4초 개선
⚠️ 아래에 첨부된 모든 코드, API 관련 정보는 예시용으로 재작성되었습니다.

## 개요

## 기여

## 구현 방식
1.	전체 메뉴 트리 데이터 조회
- `/api/menus` 엔드포인트를 통해 전체 트리를 처음 한 번에 불러오고, 브라우저 측에서 dataMap에 { menuId : 노드 정보 } 구조로 저장.
- 이 때, children 속성을 포함해서, 각 노드가 어떤 자식들을 갖고 있는지(또는 자식 정보 조회에 필요한 key)도 미리 넣어둠.
```javascript
var dataMap = new Map();
$.each(jsonData, function(index, result) {

    dataMap.set(result.id, {
            id : result.menuId,
            level : result.level,
            ...
            , children: // True or False (현재 노드의 hidden 상태와 하위 노드의 개수에 따라 조건부로 정해짐)
        })
});
return dataMap;
```
2.	하위 노드 ID만 조회하는 신규 REST API
- `/api/menus/ajax` 엔드포인트를 추가.
- 트리에서 특정 노드를 펼칠 때, 해당 노드의 직접적인 자식 노드 ID들만 빠르게 조회할 수 있음.
3.	jstree Ajax 호출 & dataFilter
- 사용자가 트리에서 화살표(펼침 버튼)를 누르면, jsTree에서 Ajax 호출로 자식 노드 ID를 가져옴.
- dataFilter 로직에서 dataMap에 이미 저장된 전체 노드 정보를 참조하여, 필요한 자식 노드(실제 데이터)만 가져와 트리에 렌더링.
```javascript
'core' : {
  'data' : {
    'url': '/api/menus/ajax.json',
    'data': function(node) {
      return {
          "par_id": node.id === '#' ? null : node.id,
      }
    },
    'dataFilter': function(data, type) {
      var parsed_data = JSON.parse(data);
      var chd_ids = parsedData.children;
      var chd_lst = [];
      Array.from(chd_ids).forEach((id) => { // options.json_data :: 이미 저장된 전체 노드 정보
        chd_lst.push(options.json_data.get(id));
      });
      return JSON.stringify(children);
    }
  },
...
}
```

<br>

## 문제 및 해결 방법
### 기존 jstree 내장 함수인 refresh에서 side effect 발생
#### 📖 배경
- 기존 로직:
    - 전체 트리 데이터를 한 번에 조회하고, jsTree의 refresh() 함수를 통해 메뉴 추가/삭제/위치 이동 등의 변경사항을 자동 반영.
- 개선 작업:
    - 최초 로딩 시간을 단축하기 위해 Ajax 기반 Lazy Loading 방식을 도입.
    - 필요한 시점에만 하위 노드를 가져오도록 변경하여 초기 렌더링 성능을 대폭 개선(15초 → 4초).
- 문제 발생:
    - Lazy Loading 로직에서 트리를 부분 로딩하게 되면서, refresh() 함수가 “전체 트리” 전제 조건을 만족하지 못함.
    - 즉, 전체 트리 JSON이 아닌 “부분 트리” 데이터만 가지고는 refresh()가 정상 동작하지 않아, 메뉴 변경 시 반영이 제대로 되지 않음.

#### 🔍 문제 원인
- jsTree의 refresh() 메커니즘:
    - 트리를 전체 구조(JSON)로 다시 읽어와야 노드 추가/삭제/갱신을 자동으로 처리.
    - Lazy Loading된 상태에서는 일부 노드만 동적으로 불러오므로, 기존 방식 그대로 refresh()를 호출해도 트리 전체를 갱신할 수 없음.

#### ✅ 해결 방법
1.	전체 트리 데이터 재조회
    - 메뉴 변경(추가/삭제/이동) 발생 시, /api/menus로 전체 트리 데이터를 다시 조회.
    - level 값(트리 구조상 필요한 레벨 정보) 등을 부분적으로만 갱신하는 것보다, 전체를 다시 불러오는 편이 구조적으로 안정적임.
2.	현재 노드 상태(열림/선택 상태) 저장
    - jsTree는 상태(state) 정보를 내부적으로 관리하지만, Lazy Loading으로 구조가 달라져 refresh() 시점에 꼬일 수 있음.
    - 해결책: 사용자가 열어 놓은 노드 목록과 선택(select)되어 있는 노드 정보를 별도로 저장(캐싱).
3.	트리 전체를 destroy()
    - 기존 트리를 완전히 파기(destroy)함으로써, 불필요한 상태나 부분적으로 바뀐 노드를 리셋.
    - 새로 가져온 전체 트리 데이터를 기반으로 jsTree를 다시 초기화.
4.	노드 열림(open_node) & 선택(select_node) 복원
    - 트리 초기화 후, 저장해 두었던 **열려 있던 노드의 상위 노드부터 차례대로 open_node**를 호출해 펼친다.
    - select되어 있던 노드가 있었다면, 해당 노드를 다시 select_node 처리하여 사용자 상태를 유지.
  
#### 📌 결론
    •	Lazy Loading으로 초기 로딩 속도를 극적으로 줄이면서도,
    •	전체 트리 변경이 필요할 때는 destroy() → open_node()/select_node() 복원을 통해 jsTree의 상태를 정상적으로 유지.
    •	사용자 입장에서는 **“메뉴 변경 후에도 문제없이 트리가 갱신”**되는 경험을 하게 됨.

#### 📝  시사점
    •	**부분 로딩(성능 개선)**과 전체 재로딩(데이터 일관성) 사이에서 트레이드오프가 존재함.
    •	jsTree 등 UI 컴포넌트를 활용할 때, 라이브러리의 refresh 메커니즘과 Lazy Loading 방식을 잘 이해해야 함.
    •	트리가 복잡하거나 연관된 로직이 많으면, 상태(state) 관리가 중요하다는 점을 깨달음.

<br>

### 기존 jstree 내장 함수인 open_node에서 side effect 발생
#### 📖 배경
#### 🔍 해결 과정
#### ✅ 해결 방법
#### 📌 결론
