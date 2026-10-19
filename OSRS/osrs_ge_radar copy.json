"""
========================================================================================
⚔️ OSRS Live Grand Exchange Radar & Margin Scanner (رادار بورصة رون سكيب اللحظي)
========================================================================================
- المرجع الفني: فيديو FlippingOldSchool + OSRS Wiki Real-Time Prices API
- الوظيفة:
  1. جلب أسعار الشراء والبيع اللحظية لـ 4600+ عنصر في بورصة Grand Exchange.
  2. حساب هامش الربح الصافي (Net Margin) بعد خصم ضريبة البورصة (GE Tax = 1% بحد أقصى 5M GP).
  3. حساب العائد على الاستثمار (ROI%) والربح الإجمالي المتوقع بناءً على حدود الشراء (Buy Limit / 4h).
  4. فرز وتصنيف الفرص: (عالية الربح للمشتركين P2P، وفرص آمنة للحسابات المجانية F2P).
========================================================================================
"""

import urllib.request
import json
import time
import os
import sys

# ضمان دعم الرموز التعبيرية واللغة العربية في كونسول ويندوز
if sys.platform == 'win32':
    sys.stdout.reconfigure(encoding='utf-8')

HEADERS = {
    'User-Agent': 'Bolla_OSRS_Flipping_Radar_v1.0 (Business Automation Research)'
}

API_LATEST = "https://prices.runescape.wiki/api/v1/osrs/latest"
API_MAPPING = "https://prices.runescape.wiki/api/v1/osrs/mapping"

def fetch_json(url):
    req = urllib.request.Request(url, headers=HEADERS)
    with urllib.request.urlopen(req, timeout=15) as res:
        return json.loads(res.read().decode('utf-8'))

def calculate_ge_tax(sell_price):
    """ضريبة البورصة 1% بحد أقصى 5,000,000 ذهبة للقطعة الواحدة"""
    if sell_price < 100:
        return 0
    tax = int(sell_price * 0.01)
    return min(tax, 5000000)

def scan_market(min_roi=1.5, min_profit=5000):
    print("⏳ جاري سحب بيانات البورصة اللحظية من OSRS Wiki API...")
    mapping_raw = fetch_json(API_MAPPING)
    mapping = {item['id']: item for item in mapping_raw}
    
    prices_raw = fetch_json(API_LATEST)
    prices = prices_raw.get('data', {})
    
    opportunities_members = []
    opportunities_f2p = []
    
    for item_id_str, p in prices.items():
        item_id = int(item_id_str)
        meta = mapping.get(item_id)
        if not meta:
            continue
            
        high = p.get('high') # سعر الشراء الفوري
        low = p.get('low')   # سعر البيع الفوري
        
        if not high or not low or high <= low:
            continue
            
        tax = calculate_ge_tax(high)
        net_margin = high - low - tax
        
        if net_margin <= 0:
            continue
            
        roi = (net_margin / low) * 100
        limit = meta.get('limit') or 1
        total_potential = net_margin * limit
        is_members = meta.get('members', True)
        
        entry = {
            'id': item_id,
            'name': meta.get('name', 'Unknown'),
            'buy_price': low,
            'sell_price': high,
            'tax': tax,
            'net_margin': net_margin,
            'limit': limit,
            'total_potential_profit': total_potential,
            'roi_pct': round(roi, 2),
            'members': is_members
        }
        
        if roi >= min_roi and total_potential >= min_profit:
            if is_members:
                opportunities_members.append(entry)
            else:
                opportunities_f2p.append(entry)
                
    # فرز حسب أعلى ربح إجمالي
    opportunities_members.sort(key=lambda x: x['total_potential_profit'], reverse=True)
    opportunities_f2p.sort(key=lambda x: x['total_potential_profit'], reverse=True)
    
    return opportunities_members, opportunities_f2p

def print_table(title, items, limit=10):
    print("\n" + "=" * 95)
    print(f"🎯 {title} (أعلى {limit} فرص ربحية الآن)")
    print("=" * 95)
    header = f"{'اسم العنصر (Item Name)':<32} | {'سعر الشراء':<11} | {'سعر البيع':<11} | {'الصافي للقطعة':<13} | {'الحد':<6} | {'الربح المتوقع':<14} | {'العائد ROI'}"
    print(header)
    print("-" * 95)
    
    for item in items[:limit]:
        name = item['name'][:30]
        buy = f"{item['buy_price']:,}"
        sell = f"{item['sell_price']:,}"
        margin = f"+{item['net_margin']:,}"
        lim = str(item['limit'])
        profit = f"{item['total_potential_profit']:,} GP"
        roi = f"{item['roi_pct']}%"
        print(f"{name:<32} | {buy:<11} | {sell:<11} | {margin:<13} | {lim:<6} | {profit:<14} | {roi}")
    print("=" * 95)

if __name__ == '__main__':
    start_time = time.time()
    members_opps, f2p_opps = scan_market(min_roi=1.0, min_profit=10000)
    elapsed = round(time.time() - start_time, 2)
    
    print(f"\n⚡ تم فحص السوق بالكامل في {elapsed} ثانية!")
    print(f"📊 إجمالي الفرص المكتشفة: {len(members_opps)} فرصة للأعضاء (P2P) و {len(f2p_opps)} فرصة للمجاني (F2P)")
    
    print_table("فرص الحسابات المجانية (F2P Opportunities - مثالية للبداية فوراً)", f2p_opps, limit=10)
    print_table("فرص الحسابات المشتركة (P2P Members Opportunities - أرباح بالملايين)", members_opps, limit=10)
    
    # حفظ النتائج في ملف JSON للاستخدام المباشر
    output_path = os.path.join(os.path.dirname(__file__), 'osrs_radar_live.json')
    with open(output_path, 'w', encoding='utf-8') as f:
        json.dump({
            'timestamp': time.strftime('%Y-%m-%d %H:%M:%S'),
            'f2p_top': f2p_opps[:25],
            'members_top': members_opps[:25]
        }, f, ensure_ascii=False, indent=2)
    print(f"\n💾 تم حفظ لقطة السوق الحية في: {output_path}")
