# -import streamlit as st
import random
import requests
from bs4 import BeautifulSoup
import pandas as pd
import time

st.set_page_config(page_title="로또 조합 생성기", page_icon="🎰", layout="centered")

st.title("🎰 로또 조합 생성기")
st.write("최근 50회차 당첨번호를 제외하고 조건 기반으로 추천 조합을 생성해요!")

# ---------------------------
# 1. 당첨 번호 크롤링 함수
# ---------------------------
@st.cache_data(show_spinner=True)
def get_latest_round():
    url = "https://www.dhlottery.co.kr/gameResult.do?method=byWin"
    resp = requests.get(url)
    soup = BeautifulSoup(resp.text, "html.parser")
    round_text = soup.select_one(".win_result h4 strong").text
    return int(round_text.replace("회", "").strip())

@st.cache_data(show_spinner=True)
def get_past_winning_numbers(n=50):
    latest = get_latest_round()
    base_url = "https://www.dhlottery.co.kr/gameResult.do?method=byWin&drwNo="
    all_numbers = set()

    for i in range(latest, latest - n, -1):
        res = requests.get(base_url + str(i))
        soup = BeautifulSoup(res.text, "html.parser")
        nums = soup.select(".num.win p span")
        numbers = [int(n.text.strip()) for n in nums]
        all_numbers.update(numbers)
        time.sleep(0.1)

    return all_numbers

# ---------------------------
# 2. 유효한 조합 필터링
# ---------------------------
def is_valid_combination(combo):
    odds = sum(1 for n in combo if n % 2 != 0)
    evens = 6 - odds
    if not (2 <= odds <= 4):
        return False

    z1 = sum(1 for n in combo if 1 <= n <= 15)
    z2 = sum(1 for n in combo if 16 <= n <= 30)
    z3 = sum(1 for n in combo if 31 <= n <= 45)
    if min(z1, z2, z3) == 0:
        return False

    if any(combo[i] + 1 == combo[i + 1] for i in range(5)):
        return False

    return True

# ---------------------------
# 3. 조합 생성
# ---------------------------
def generate_lotto(exclude_nums, count=5):
    results = []
    tries = 0

    while len(results) < count:
        tries += 1
        pool = [n for n in range(1, 46) if n not in exclude_nums]
        combo = sorted(random.sample(pool, 6))

        if is_valid_combination(combo):
            results.append(combo)

    return results

# ---------------------------
# 실행 & UI
# ---------------------------
if st.button("✨ 추천 조합 생성"):
    st.info("로딩 중... 당첨 번호 수집 및 조합 생성 중입니다.")
    exclude_nums = get_past_winning_numbers()
    combos = generate_lotto(exclude_nums, count=5)

    df = pd.DataFrame(combos, columns=["번호1", "번호2", "번호3", "번호4", "번호5", "번호6"])
    st.success("추천 조합입니다!")
    st.dataframe(df, use_container_width=True)

    # 다운로드 버튼
    csv = df.to_csv(index=False).encode("utf-8-sig")
    st.download_button("⬇️ 엑셀 다운로드", csv, "추천_로또조합.csv", "text/csv")
