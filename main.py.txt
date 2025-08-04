from flask import Flask, request, jsonify
from collections import defaultdict

app = Flask(__name__)

@app.route('/distribute', methods=['POST'])
def distribute_tips():
    data = request.json

    employees = data['employees']  # list of dicts: {'name': str, 'amount': int}
    bills = {int(k): v for k, v in data['bills'].items()}
    coins = {int(k): v for k, v in data['coins'].items()}
    initial_cash = data['initial_cash']
    min_cash_to_leave = data.get('min_cash', 2500)

    # כלל כל הקופה
    total_cash_available = sum([k * v for k, v in {**bills, **coins}.items()])
    distributable_cash = initial_cash + total_cash_available - min_cash_to_leave
    total_needed = sum(emp['amount'] for emp in employees)

    if distributable_cash < total_needed:
        return jsonify({'error': 'Not enough cash to distribute'}), 400

    # עדיפות לפרוס קודם כל שטרות גדולים, אח"כ מטבעות
    denominations = sorted(list(bills.keys()) + list(coins.keys()), reverse=True)

    # מצב נוכחי של הקופה - שטרות ומטבעות
    cash_pool = defaultdict(int, {**bills, **coins})

    results = []
    overpaid = []
    underpaid = []

    for emp in sorted(employees, key=lambda e: e['amount'], reverse=True):
        name = emp['name']
        amount_needed = emp['amount']
        cash_given = 0
        given_bills = defaultdict(int)

        for denom in denominations:
            while denom <= (amount_needed - cash_given) and cash_pool[denom] > 0:
                cash_pool[denom] -= 1
                given_bills[denom] += 1
                cash_given += denom

        delta = cash_given - amount_needed
        if delta > 0:
            overpaid.append({'name': name, 'extra': delta})
        elif delta < 0:
            underpaid.append({'name': name, 'missing': -delta})

        results.append({
            'name': name,
            'total': amount_needed,
            'cash_given': cash_given,
            'bills': {str(k): v for k, v in given_bills.items() if k >= 20},
            'coins': {str(k): v for k, v in given_bills.items() if k < 20},
            'bit_needed': max(0, amount_needed - cash_given),
            'bit_to_send': max(0, cash_given - amount_needed)
        })

    # ביט: העברות בין עובדים
    transfers = []
    i = 0
    j = 0

    while i < len(overpaid) and j < len(underpaid):
        sender = overpaid[i]
        receiver = underpaid[j]

        transfer_amount = min(sender['extra'], receiver['missing'])
        transfers.append({
            'from': sender['name'],
            'to': receiver['name'],
            'amount': transfer_amount
        })

        sender['extra'] -= transfer_amount
        receiver['missing'] -= transfer_amount

        if sender['extra'] == 0:
            i += 1
        if receiver['missing'] == 0:
            j += 1

    return jsonify({
        'status': 'ok',
        'summary': {
            'total_cash_distributed': sum(r['cash_given'] for r in results),
            'bit_total': sum(r['bit_needed'] for r in results),
            'remaining_in_register': min_cash_to_leave + sum(k * v for k, v in cash_pool.items())
        },
        'employees': results,
        'bit_transfers': transfers
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
