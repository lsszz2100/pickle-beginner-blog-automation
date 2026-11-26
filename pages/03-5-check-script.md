# 03-5. 검수 스크립트 만들기

발행 전에 원고 한 편을 검사하는 스크립트를 만듭니다. 이 스크립트를 통과하지 못한 원고는 발행 절차로 넘어가지 않습니다.

## 검사 항목

| 항목 | 조건 | 실패 시 |
| --- | --- | --- |
| 글자 수 | 3,000~5,000자 | 발행 중단 |
| 섹션 | h1이 5~7개 | 발행 중단 |
| 소제목 | h2가 1개 이상 | 경고 |
| 이미지 | img가 3~7개 | 발행 중단 |
| 태그 | 정확히 10개 | 발행 중단 |
| 제목 | 10~40자 | 경고 |
| 금지어 | 지정한 표현 미포함 | 경고 |
| 출처 표기 | 이미지가 있으면 한 줄 | 경고 |

## 스크립트

```python
#!/usr/bin/env python3
import json, re, sys, html
from pathlib import Path

SPEC = {
    "min_chars": 3000, "max_chars": 5000,
    "min_images": 3, "max_images": 7,
    "tags": 10, "min_h1": 5, "max_h1": 7,
    "banned": ["완벽 가이드", "완전 정복", "궁극의"],
}

def text_of(doc):
    t = re.sub(r"<script.*?</script>", " ", doc, flags=re.S)
    t = re.sub(r"<[^>]+>", " ", t)
    return re.sub(r"\s+", " ", html.unescape(t)).strip()

def check(body_path):
    body = Path(body_path).read_text(encoding="utf-8")
    meta_path = Path(body_path).with_suffix(".meta.json")
    meta = json.loads(meta_path.read_text(encoding="utf-8")) if meta_path.exists() else {}
    text = text_of(body)
    errors, warnings = [], []

    n = len(text)
    if not SPEC["min_chars"] <= n <= SPEC["max_chars"]:
        errors.append(f"본문 {n}자 (기준 {SPEC['min_chars']}~{SPEC['max_chars']})")

    h1 = len(re.findall(r"<h1[ >]", body, flags=re.I))
    if not SPEC["min_h1"] <= h1 <= SPEC["max_h1"]:
        errors.append(f"h1 {h1}개 (기준 {SPEC['min_h1']}~{SPEC['max_h1']})")

    images = re.findall(r"<img[^>]+src=[\"']([^\"']+)", body, flags=re.I)
    if not SPEC["min_images"] <= len(images) <= SPEC["max_images"]:
        errors.append(f"이미지 {len(images)}장 (기준 {SPEC['min_images']}~{SPEC['max_images']})")

    tags = [t.strip() for t in meta.get("tags", []) if t.strip()]
    if len(tags) != SPEC["tags"]:
        errors.append(f"태그 {len(tags)}개 (기준 {SPEC['tags']})")
    if len(set(tags)) != len(tags):
        errors.append("중복 태그가 있습니다")

    title = meta.get("title", "")
    if not 10 <= len(title) <= 40:
        warnings.append(f"제목 {len(title)}자")

    for word in SPEC["banned"]:
        if word in text or word in title:
            warnings.append(f"금지어 포함: {word}")

    if images and "이미지 출처" not in text:
        warnings.append("이미지 출처 표기가 없습니다")

    return errors, warnings

if __name__ == "__main__":
    total = 0
    for path in sys.argv[1:]:
        errors, warnings = check(path)
        print(f"[{Path(path).name}]")
        for e in errors:
            print("  실패:", e)
        for w in warnings:
            print("  경고:", w)
        if not errors:
            print("  통과")
        total += len(errors)
    sys.exit(1 if total else 0)
```

## 실행

```bash
python3 check_post.py gen/01.html gen/02.html gen/03.html
echo "exit: $?"
```

실패가 하나라도 있으면 종료 코드가 1이 되고, 발행 스크립트는 그 배치를 시작하지 않습니다. 이 종료 코드가 자동화의 안전장치입니다. 사람이 읽지 않아도 규격 위반은 발행되지 않습니다.

## 경고와 실패를 나누는 이유

경고는 발행을 막지 않습니다. 제목이 12자여도 검색에 실릴 수 있고, 출처 표기가 빠져도 글이 깨지지 않습니다. 반면 태그가 아홉 개인 글은 나중에 태그 체계를 흐트러뜨립니다. 판단이 서지 않는 항목은 경고로 두고, 실제로 문제를 일으킨 항목만 실패로 올립니다. 처음부터 전부 실패로 잡으면 통과하는 원고가 없어 절차를 끄게 됩니다.
