import streamlit as st
import pandas as pd
import plotly.graph_objects as go
from plotly.subplots import make_subplots
from openai import OpenAI
import io

# --- 1. 页面配置 ---
st.set_page_config(page_title="亚马逊竞品全生命周期追踪系统", layout="wide", initial_sidebar_state="expanded")

st.title("📊 亚马逊竞品追踪与智能分析系统")
st.markdown("该系统支持多份竞品汇总表合并分析，通过 AI 专家模型深度复盘对手运营套路。")

# --- 2. 侧边栏：配置中心 ---
with st.sidebar:
    st.header("⚙️ 配置中心")
    api_key = st.text_input("输入 DeepSeek API Key", type="password", help="用于生成 AI 深度分析报告")
    model = st.selectbox("选择 AI 模型", ["deepseek-chat", "deepseek-reasoner"], index=0)
    handle_zero = True


# --- 3. 核心数据处理逻辑 ---
def process_amazon_data(uploaded_files):
    all_dfs = []
    for file in uploaded_files:
        try:
            if file.name.endswith('.xlsx') or file.name.endswith('.xls'):
                df = pd.read_excel(file, dtype=object)
            else:
                content = file.getvalue()
                # 尝试多种编码读取 CSV
                for enc in ['utf-8', 'gbk', 'gb18030']:
                    try:
                        df = pd.read_csv(io.BytesIO(content), encoding=enc, dtype=str)
                        break
                    except:
                        continue

            # 标准化处理列名
            df.columns = [str(c).strip() for c in df.columns]
            if 'ASIN' not in df.columns or '类型' not in df.columns:
                st.warning(f"文件 {file.name} 格式不符（缺少 ASIN 或 类型 列），已跳过")
                continue

            # 逆透视：从日期列横向转为纵向
            base_cols = ['ASIN', '类型']
            date_cols = [c for c in df.columns if c not in base_cols]
            df_melted = df.melt(id_vars=base_cols, value_vars=date_cols, var_name='日期', value_name='数值')
            all_dfs.append(df_melted)
        except Exception as e:
            st.error(f"处理文件 {file.name} 时出错: {e}")

    if not all_dfs: return None

    # 合并多文件数据
    final_df = pd.concat(all_dfs)

    # 透视表：将“类型”中的 销量、价格 等转为列
    pivot_df = final_df.pivot_table(index=['ASIN', '日期'], columns='类型', values='数值',
                                    aggfunc='first').reset_index()

    # 转换日期格式
    pivot_df['日期'] = pd.to_datetime(pivot_df['日期'])

    # 还原数值类型以便计算和绘图
    numeric_cols = ['销量', '最终价格', 'Rating数', '大类排名', '小类排名', '评分', 'BD次数', 'LD次数', 'Coupon金额',
                    'BD价格', 'LD价格']
    for col in numeric_cols:
        if col in pivot_df.columns:
            pivot_df[col] = pd.to_numeric(pivot_df[col], errors='coerce')
            if handle_zero:
                # 将 0 替换为 NA，Plotly 绘图时会自动断开连线
                pivot_df[col] = pivot_df[col].replace(0, pd.NA)

    return pivot_df.sort_values(['ASIN', '日期'])


# --- 4. 文件上传与分析主界面 ---
uploaded_files = st.file_uploader("第一步：批量上传竞品趋势表 (.xlsx, .csv)", accept_multiple_files=True,
                                  type=['xlsx', 'xls', 'csv'])

if uploaded_files:
    df = process_amazon_data(uploaded_files)

    if df is not None:
        asins = df['ASIN'].unique().tolist()
        st.success(f"✅ 成功加载 {len(asins)} 个竞品数据")

        # 让用户选择要对比的竞品
        selected_asins = st.multiselect("第二步：选择要对比分析的 ASIN", asins, default=asins)

        if selected_asins:
            filtered_df = df[df['ASIN'].isin(selected_asins)]

            # --- 5. 可视化部分 (优化后的分拆版) ---
            st.divider()

            # 5.1 销量趋势图
            fig_sales = go.Figure()
            for asin in selected_asins:
                asin_df = filtered_df[filtered_df['ASIN'] == asin]
                fig_sales.add_trace(go.Scatter(
                    x=asin_df['日期'], y=asin_df['销量'],
                    name=f"{asin} 销量", mode='lines+markers',
                    marker=dict(size=4), line=dict(width=2)
                ))
            fig_sales.update_layout(title="📈 日销量趋势 (Daily Sales)", xaxis_title="日期", yaxis_title="销量 (Units)",
                                    hovermode="x unified")
            st.plotly_chart(fig_sales, use_container_width=True)

            # 5.2 价格趋势图
            fig_price = go.Figure()
            for asin in selected_asins:
                asin_df = filtered_df[filtered_df['ASIN'] == asin]
                if '最终价格' in asin_df.columns:
                    fig_price.add_trace(go.Scatter(
                        x=asin_df['日期'], y=asin_df['最终价格'],
                        name=f"{asin} 价格", mode='lines',
                        line=dict(dash='dash')
                    ))
            fig_price.update_layout(title="💰 最终价格历史 (Price History)", xaxis_title="日期", yaxis_title="价格 ($)",
                                    hovermode="x unified")
            st.plotly_chart(fig_price, use_container_width=True)

            # 5.3 促销活动节奏专家看板 (增强版)
            st.subheader("⭐ 促销活动节奏看板")
            fig_act = go.Figure()

            for asin in selected_asins:
                asin_df = filtered_df[filtered_df['ASIN'] == asin].copy()

                # BD 活动 (金色星星)
                if 'BD次数' in asin_df.columns:
                    bd_acts = asin_df[asin_df['BD次数'].fillna(0) > 0]
                    if not bd_acts.empty:
                        fig_act.add_trace(go.Scatter(
                            x=bd_acts['日期'], y=[asin] * len(bd_acts),
                            mode='markers', name=f"{asin} BD (周秒杀)",
                            marker=dict(size=14, symbol='star', color='gold', line=dict(width=1, color='black')),
                            text=[f"BD价格: ${p}" for p in bd_acts.get('BD价格', bd_acts['最终价格'])],
                            hovertemplate="<b>%{text}</b><br>日期: %{x}<extra></extra>"
                        ))

                # LD 活动 (青色菱形)
                if 'LD次数' in asin_df.columns:
                    ld_acts = asin_df[asin_df['LD次数'].fillna(0) > 0]
                    if not ld_acts.empty:
                        fig_act.add_trace(go.Scatter(
                            x=ld_acts['日期'], y=[asin] * len(ld_acts),
                            mode='markers', name=f"{asin} LD (秒杀)",
                            marker=dict(size=10, symbol='diamond', color='cyan', line=dict(width=1, color='darkblue')),
                            text=[f"LD价格: ${p}" for p in ld_acts.get('LD价格', ld_acts['最终价格'])],
                            hovertemplate="<b>%{text}</b><br>日期: %{x}<extra></extra>"
                        ))

            fig_act.update_layout(
                title="促销活动分布 (金色星: BD / 青色菱形: LD)",
                xaxis_title="日期", yaxis_title="ASIN",
                yaxis=dict(autorange="reversed"),
                height=300 + (len(selected_asins) * 40),
                showlegend=True
            )
            st.plotly_chart(fig_act, use_container_width=True)

            # --- 6. AI 深度报告生成 ---
            st.divider()
            st.subheader("🧠 专家级竞品深度分析报告")

            if st.button("🚀 生成智能深度分析报告"):
                if not api_key:
                    st.error("请先在左侧输入 DeepSeek API Key")
                else:
                    # 准备数据摘要
                    analysis_context = ""
                    for asin in selected_asins:
                        asin_data = filtered_df[filtered_df['ASIN'] == asin]
                        recent_30 = asin_data.tail(30)  # 核心分析近30天

                        avg_sales = recent_30['销量'].mean()
                        max_sales = recent_30['销量'].max()
                        price_min = recent_30['最终价格'].min() if '最终价格' in recent_30.columns else 0
                        price_max = recent_30['最终价格'].max() if '最终价格' in recent_30.columns else 0
                        avg_bsr = recent_30['小类排名'].mean() if '小类排名' in recent_30.columns else 0

                        # 兼容不同列名
                        rev_col = 'Rating数' if 'Rating数' in recent_30.columns else 'Rating_num'
                        review_total = recent_30[rev_col].iloc[-1] if rev_col in recent_30.columns else "N/A"
                        current_rating = recent_30['评分'].iloc[-1] if '评分' in recent_30.columns else "N/A"

                        promo_days = (recent_30.get('BD次数', 0).fillna(0) + recent_30.get('LD次数', 0).fillna(0)).sum()

                        analysis_context += f"""
                        ASIN: {asin}
                        - 核心数据：日均销量 {avg_sales:.1f}, 峰值日销 {max_sales:.1f}, 价格区间 ${price_min} - ${price_max}
                        - 排名表现：小类排名均值 {avg_bsr:.1f}
                        - 评价画像：评分 {current_rating}, 总评价数 {review_total}
                        - 促销频率：近30天活动天数 {promo_days}天
                        """

                    system_prompt = """
                    你是一位拥有10年经验的亚马逊资深运营总监。你的任务是根据竞品数据，撰写一份极致专业的《竞品趋势深度分析报告》。
                    报告结构要求：
                    1. 分析概述：简述背景。
                    2. 竞品核心数据总览：使用 Markdown 表格汇总 ASIN、均销、价格带、排名、促销频率及“市场定位”。
                    3. 单竞品深度解析：剖析每个 ASIN 的运营动作（如：低价冲排、BD驱动型增长等）。
                    4. 综合维度对比分析：销量梯队、价格带分布、促销效果对比。
                    5. 核心结论与运营优化建议：给出行之有效的定价、促销、排名优化策略。
                    语气要求：严谨、犀利、实战导向。
                    """

                    client = OpenAI(api_key=api_key, base_url="https://api.deepseek.com")
                    try:
                        with st.spinner('AI 专家正在深度拆解对手套路并撰写报告...'):
                            response = client.chat.completions.create(
                                model=model,
                                messages=[
                                    {"role": "system", "content": system_prompt},
                                    {"role": "user", "content": f"数据摘要：\n{analysis_context}"}
                                ],
                                temperature=0.3
                            )
                        report_content = response.choices[0].message.content
                        st.markdown(report_content)

                        st.download_button(
                            label="📥 导出分析报告 (.txt)",
                            data=report_content,
                            file_name=f"竞品分析报告_{pd.Timestamp.now().strftime('%m%d')}.txt",
                            mime="text/plain"
                        )
                    except Exception as e:
                        st.error(f"AI 分析失败: {e}")
else:
    st.info("👋 欢迎使用竞品分析系统！请在上方上传 Excel 或 CSV 数据文件开始。")
