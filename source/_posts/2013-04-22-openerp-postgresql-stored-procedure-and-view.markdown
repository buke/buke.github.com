---
layout: post
title: "openerp 使用 postgresql 存储过程和视图"
date: 2013-04-22 13:21
comments: true
categories: OpenERP
---

OpenERP 使用 postgresql 存储过程和试图，步骤如下：

STEP1: 在模块的 init 函数中定义存储过程

{% codeblock lang:python %}
    def init(self, cr):
        ''' create stored procedure '''
        cr.execute("""CREATE OR REPLACE FUNCTION fn_fi_report_childs(int)
        RETURNS TABLE(id int) AS $$
            WITH RECURSIVE t AS (
                SELECT id,parent_id  FROM fi_report WHERE id = $1
              UNION ALL
                SELECT fi_report.id, fi_report.parent_id FROM fi_report, t WHERE fi_report.parent_id = t.id
            )
            SELECT id FROM t;
        $$ LANGUAGE SQL
            """)
{% endcodeblock %}

或者定义视图
{% codeblock lang:python %}
    def init(self, cr):
        tools.drop_view_if_exists(cr, 'analytic_entries_report')
        cr.execute("""
            create or replace view analytic_entries_report as (
                 select
                     min(a.id) as id,
                     count(distinct a.id) as nbr,
                     a.date as date,
                     to_char(a.date, 'YYYY') as year,
                     to_char(a.date, 'MM') as month,
                     to_char(a.date, 'YYYY-MM-DD') as day,
                     a.user_id as user_id,
                     a.name as name,
                     analytic.partner_id as partner_id,
                     a.company_id as company_id,
                     a.currency_id as currency_id,
                     a.account_id as account_id,
                     a.general_account_id as general_account_id,
                     a.journal_id as journal_id,
                     a.move_id as move_id,
                     a.product_id as product_id,
                     a.product_uom_id as product_uom_id,
                     sum(a.amount) as amount,
                     sum(a.unit_amount) as unit_amount
                 from
                     account_analytic_line a, account_analytic_account analytic
                 where analytic.id = a.account_id
                 group by
                     a.date, a.user_id,a.name,analytic.partner_id,a.company_id,a.currency_id,
                     a.account_id,a.general_account_id,a.journal_id,
                     a.move_id,a.product_id,a.product_uom_id
            )
        """)
{% endcodeblock %}


STEP2: 在模块的函数中使用存储过程

{% codeblock lang:python %}
    def get_amount(self,cr,uid,id,period_id,context=None):
        cr.execute('SELECT * FROM fn_fi_report_childs(%s)', (id,))
{% endcodeblock %}

而视图的话，则如普通的表一样使用。


STEP3: 完成！ 

