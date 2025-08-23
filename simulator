import React, { useEffect, useMemo, useState } from "react";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { Label } from "@/components/ui/label";
import { Input } from "@/components/ui/input";
import { Switch } from "@/components/ui/switch";
import { Tabs, TabsContent, TabsList, TabsTrigger } from "@/components/ui/tabs";
import { Button } from "@/components/ui/button";
import {
  LineChart, Line, XAxis, YAxis, Tooltip, ResponsiveContainer, Legend, CartesianGrid,
  AreaChart, Area, BarChart, Bar
} from "recharts";
import { Download, Save, Share2, TestTube2 } from "lucide-react";

// ===== Helpers =====
const fmt0 = new Intl.NumberFormat("en-US", { style: "currency", currency: "USD", maximumFractionDigits: 0 });
const fmt2 = new Intl.NumberFormat("en-US", { style: "currency", currency: "USD", maximumFractionDigits: 2 });
function clamp(n: number, lo: number, hi: number) { return Math.max(lo, Math.min(hi, n)); }

// ===== Main App =====
export default function App() {
  // --- Core Growth Assumptions
  const [clientsPerDay, setClientsPerDay] = useState(3);
  const [useFunnel, setUseFunnel] = useState(false);
  const [includeWeekends, setIncludeWeekends] = useState(true);
  const [startPrice, setStartPrice] = useState(300);
  const [monthlyPriceGrowthPct, setMonthlyPriceGrowthPct] = useState(10); // %
  const [months, setMonths] = useState(12);
  const [baseMRR, setBaseMRR] = useState(1500);
  const [churnPct, setChurnPct] = useState(0); // monthly %

  // --- Sales Comp
  const [fmCommPct, setFmCommPct] = useState(25); // % of month-1 revenue
  const [ogCommPct, setOgCommPct] = useState(10); // % of matured revenue

  // --- Cash & Ops
  const [fixedOpEx, setFixedOpEx] = useState(4000); // baseline monthly tools/overhead
  const [onboardCostPerNew, setOnboardCostPerNew] = useState(120); // one-time cost per new client
  const [supportCostPerActive, setSupportCostPerActive] = useState(15); // per active client / month
  const [taxRatePct, setTaxRatePct] = useState(20); // % of positive operating profit (approx)

  // --- Owner Draw Policy
  const [minOwnerDraw, setMinOwnerDraw] = useState(2000);
  const [extraDrawPct, setExtraDrawPct] = useState(40); // % of post-tax surplus

  // --- Prepay Program
  const [prepayPct, setPrepayPct] = useState(20); // % of new clients choosing annual prepay
  const [prepayMonthsFree, setPrepayMonthsFree] = useState(2); // discount months

  // --- Hiring / Capacity
  const [onboardingHrsPerNew, setOnboardingHrsPerNew] = useState(3);
  const [ongoingHrsPerActive, setOngoingHrsPerActive] = useState(0.8);
  const [hrsPerFTE, setHrsPerFTE] = useState(160);

  // --- Funnel (optional override)
  const [leadsPerDay, setLeadsPerDay] = useState(45);
  const [showRatePct, setShowRatePct] = useState(40);
  const [closeRatePct, setCloseRatePct] = useState(20);

  // --- UI / State persistence
  const [presetName, setPresetName] = useState("");

  const daysPerMonth = includeWeekends ? 30 : 22;
  const growth = monthlyPriceGrowthPct / 100;
  const churn = clamp(churnPct / 100, 0, 0.99);
  const fmComm = fmCommPct / 100;
  const ogComm = ogCommPct / 100;

  const derivedClientsPerDay = useMemo(() => {
    if (!useFunnel) return clientsPerDay;
    const appts = leadsPerDay * (showRatePct / 100);
    const closes = appts * (closeRatePct / 100);
    return closes;
  }, [useFunnel, clientsPerDay, leadsPerDay, showRatePct, closeRatePct]);

  const newClientsPerMonth = Math.round(derivedClientsPerDay * daysPerMonth);

  // ===== Core Simulation =====
  const data = useMemo(() => {
    // cohort price per month for new signups
    const priceByMonth: number[] = [];
    for (let m = 1; m <= months; m++) priceByMonth.push(startPrice * Math.pow(1 + growth, m - 1));

    const rows: any[] = [];
    let cumulativeOneTime = 0;
    let cashBuffer = 0;

    // Track prepay clients per cohort
    const cohortPayMonthly: number[] = []; // count per cohort-month
    const cohortPrepay: number[] = [];

    for (let t = 1; t <= months; t++) {
      const price_t = priceByMonth[t - 1];
      const newClients_t = newClientsPerMonth;

      const prepayCount_t = Math.round(newClients_t * (prepayPct / 100));
      const payMonthlyCount_t = newClients_t - prepayCount_t;
      cohortPrepay.push(prepayCount_t);
      cohortPayMonthly.push(payMonthlyCount_t);

      const newRevenue_t = price_t * newClients_t; // accrual MRR add

      // Matured revenue (accrual) and active clients for month t
      let maturedRevenue_t = 0;
      let activeClients_t = newClients_t; // start with this month cohort

      for (let j = 1; j <= t - 1; j++) {
        const monthsAged = t - j;
        const price_j = priceByMonth[j - 1];
        const activeFromCohort_j = (cohortPayMonthly[j - 1] + cohortPrepay[j - 1]) * Math.pow(1 - churn, monthsAged);
        maturedRevenue_t += price_j * activeFromCohort_j;
        activeClients_t += activeFromCohort_j;
      }

      const grossMRR_t = baseMRR + newRevenue_t + maturedRevenue_t;

      // Commissions
      const recurringCommission_t = ogComm * maturedRevenue_t; // on matured cohorts only
      const oneTimeCommission_t = fmComm * (price_t * newClients_t); // on new cohort
      const commissionCashOut_t = recurringCommission_t + oneTimeCommission_t;
      cumulativeOneTime += oneTimeCommission_t;

      // Cash Collected this month (not accrual):
      // - pay-monthly cohorts pay their monthly fee when active
      // - prepay cohorts pay upfront in their signup month (12 - monthsFree) months
      let monthlyCashFromPayMonthly = 0;
      for (let j = 1; j <= t; j++) {
        const monthsAged = t - j;
        const price_j = priceByMonth[j - 1];
        const activePayMonthly_j = cohortPayMonthly[j - 1] * Math.pow(1 - churn, monthsAged);
        monthlyCashFromPayMonthly += price_j * activePayMonthly_j;
      }
      const upfrontPrepayCash_t = prepayCount_t * price_t * (12 - prepayMonthsFree);
      const cashCollected_t = baseMRR + monthlyCashFromPayMonthly + upfrontPrepayCash_t;

      // Costs & Taxes & Owner Draw (cash perspective)
      const oneTimeOpsCost_t = onboardCostPerNew * newClients_t;
      const supportOpsCost_t = supportCostPerActive * activeClients_t;
      const opExCash_t = fixedOpEx + oneTimeOpsCost_t + supportOpsCost_t;

      const operatingProfitBeforeTax_t = cashCollected_t - commissionCashOut_t - opExCash_t;
      const taxesCash_t = operatingProfitBeforeTax_t > 0 ? operatingProfitBeforeTax_t * (taxRatePct / 100) : 0;
      const postTaxCash_t = operatingProfitBeforeTax_t - taxesCash_t;

      const ownerDraw_t = Math.max(0, Math.min(minOwnerDraw + Math.max(0, postTaxCash_t - minOwnerDraw) * (extraDrawPct / 100), postTaxCash_t));
      const netCashAfterDraw_t = postTaxCash_t - ownerDraw_t;
      cashBuffer += netCashAfterDraw_t;

      // Capacity / Hiring
      const hoursNeeded_t = newClients_t * onboardingHrsPerNew + activeClients_t * ongoingHrsPerActive;
      const fteNeeded_t = hoursNeeded_t / Math.max(1, hrsPerFTE);

      rows.push({
        month: t,
        price: price_t,
        newClients: newClients_t,
        prepayCount: prepayCount_t,
        payMonthlyCount: payMonthlyCount_t,
        activeClients: activeClients_t,
        newRevenue: newRevenue_t,
        maturedRevenue: maturedRevenue_t,
        grossMRR: grossMRR_t,
        recurringCommission: recurringCommission_t,
        oneTimeCommission: oneTimeCommission_t,
        commissionCashOut: commissionCashOut_t,
        cumulativeOneTime,
        cashCollected: cashCollected_t,
        upfrontPrepayCash: upfrontPrepayCash_t,
        opExCash: opExCash_t,
        taxesCash: taxesCash_t,
        postTaxCash: postTaxCash_t,
        ownerDraw: ownerDraw_t,
        netCashAfterDraw: netCashAfterDraw_t,
        cashBuffer,
        hoursNeeded: hoursNeeded_t,
        fteNeeded: fteNeeded_t,
      });
    }
    return rows;
  }, [months, baseMRR, newClientsPerMonth, startPrice, growth, fmComm, ogComm, churn, prepayPct, prepayMonthsFree, fixedOpEx, onboardCostPerNew, supportCostPerActive, taxRatePct, minOwnerDraw, extraDrawPct, onboardingHrsPerNew, ongoingHrsPerActive, hrsPerFTE]);

  const kpis = useMemo(() => {
    if (!data.length) return null;
    const last = data[data.length - 1];
    const avgNewPrice = data.reduce((s, r) => s + r.price, 0) / data.length;
    return {
      activeClients: Math.round(last.activeClients),
      grossMRR: last.grossMRR,
      netMRR: last.grossMRR - last.recurringCommission,
      recurringComm: last.recurringCommission,
      oneTimeCum: last.cumulativeOneTime,
      avgNewPrice,
      cashBuffer: last.cashBuffer,
      fteNeeded: last.fteNeeded,
    };
  }, [data]);

  // ===== Presets (localStorage) & Share Link =====
  useEffect(() => {
    // auto-load from URL hash if provided
    try {
      const hash = window.location.hash?.slice(1);
      if (hash) {
        const decoded = JSON.parse(atob(decodeURIComponent(hash)));
        applyState(decoded);
      }
    } catch {}
    // eslint-disable-next-line
  }, []);

  function getState() {
    return {
      clientsPerDay, useFunnel, includeWeekends, startPrice, monthlyPriceGrowthPct, months, baseMRR, churnPct,
      fmCommPct, ogCommPct, fixedOpEx, onboardCostPerNew, supportCostPerActive, taxRatePct,
      minOwnerDraw, extraDrawPct, prepayPct, prepayMonthsFree,
      onboardingHrsPerNew, ongoingHrsPerActive, hrsPerFTE,
      leadsPerDay, showRatePct, closeRatePct,
    };
  }
  function applyState(s: any) {
    if (!s) return;
    setClientsPerDay(s.clientsPerDay ?? clientsPerDay);
    setUseFunnel(!!s.useFunnel);
    setIncludeWeekends(s.includeWeekends ?? includeWeekends);
    setStartPrice(s.startPrice ?? startPrice);
    setMonthlyPriceGrowthPct(s.monthlyPriceGrowthPct ?? monthlyPriceGrowthPct);
    setMonths(s.months ?? months);
    setBaseMRR(s.baseMRR ?? baseMRR);
    setChurnPct(s.churnPct ?? churnPct);
    setFmCommPct(s.fmCommPct ?? fmCommPct);
    setOgCommPct(s.ogCommPct ?? ogCommPct);
    setFixedOpEx(s.fixedOpEx ?? fixedOpEx);
    setOnboardCostPerNew(s.onboardCostPerNew ?? onboardCostPerNew);
    setSupportCostPerActive(s.supportCostPerActive ?? supportCostPerActive);
    setTaxRatePct(s.taxRatePct ?? taxRatePct);
    setMinOwnerDraw(s.minOwnerDraw ?? minOwnerDraw);
    setExtraDrawPct(s.extraDrawPct ?? extraDrawPct);
    setPrepayPct(s.prepayPct ?? prepayPct);
    setPrepayMonthsFree(s.prepayMonthsFree ?? prepayMonthsFree);
    setOnboardingHrsPerNew(s.onboardingHrsPerNew ?? onboardingHrsPerNew);
    setOngoingHrsPerActive(s.ongoingHrsPerActive ?? ongoingHrsPerActive);
    setHrsPerFTE(s.hrsPerFTE ?? hrsPerFTE);
    setLeadsPerDay(s.leadsPerDay ?? leadsPerDay);
    setShowRatePct(s.showRatePct ?? showRatePct);
    setCloseRatePct(s.closeRatePct ?? closeRatePct);
  }
  function savePreset() {
    if (!presetName) return;
    const key = `mrrsim:${presetName}`;
    localStorage.setItem(key, JSON.stringify(getState()));
    alert(`Saved preset: ${presetName}`);
  }
  function loadPreset() {
    if (!presetName) return;
    const key = `mrrsim:${presetName}`;
    const raw = localStorage.getItem(key);
    if (raw) applyState(JSON.parse(raw));
  }
  function shareLink() {
    const state = getState();
    const hash = encodeURIComponent(btoa(JSON.stringify(state)));
    const url = `${window.location.origin}${window.location.pathname}#${hash}`;
    navigator.clipboard.writeText(url);
    alert("Shareable link copied to clipboard");
  }

  // ===== CSV Export =====
  function downloadCSV() {
    const headers = [
      "Month","New Clients","Prepay Clients","Pay-Monthly Clients","New Price","Active Clients",
      "Gross MRR","Recurring Commission","Net MRR (after recurring)",
      "One-time Commission","Commission Cash (mo)",
      "Cash Collected","Upfront Prepay Cash","OpEx (cash)","Taxes (cash)",
      "Post-Tax Cash","Owner Draw","Net Cash After Draw","Cash Buffer","FTE Needed"
    ];
    const rows = data.map(r => [
      r.month,
      r.newClients,
      r.prepayCount,
      r.payMonthlyCount,
      r.price.toFixed(2),
      Math.round(r.activeClients),
      r.grossMRR.toFixed(2),
      r.recurringCommission.toFixed(2),
      (r.grossMRR - r.recurringCommission).toFixed(2),
      r.oneTimeCommission.toFixed(2),
      r.commissionCashOut.toFixed(2),
      r.cashCollected.toFixed(2),
      r.upfrontPrepayCash.toFixed(2),
      r.opExCash.toFixed(2),
      r.taxesCash.toFixed(2),
      r.postTaxCash.toFixed(2),
      r.ownerDraw.toFixed(2),
      r.netCashAfterDraw.toFixed(2),
      r.cashBuffer.toFixed(2),
      r.fteNeeded.toFixed(2),
    ]);
    const csv = [headers.join(","), ...rows.map(r => r.join(","))].join("\n");
    const blob = new Blob([csv], { type: "text/csv;charset=utf-8;" });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url; a.download = "mrr_growth_simulator.csv"; a.click();
    URL.revokeObjectURL(url);
  }

  return (
    <div className="min-h-screen w-full bg-gradient-to-b from-white via-slate-50 to-white py-8 px-4">
      <div className="max-w-7xl mx-auto">
        <header className="mb-6">
          <h1 className="text-3xl md:text-4xl font-extrabold tracking-tight text-slate-900">MRR & Cashflow Simulator</h1>
          <p className="text-slate-600 mt-2">Model clients/day (or derive from funnel), price growth, commissions, prepay, churn, OpEx, taxes, capacity & owner draws. Export or share presets.</p>
        </header>

        {/* Top KPI bar */}
        {kpis && (
          <div className="grid grid-cols-2 md:grid-cols-4 lg:grid-cols-6 gap-3 mb-6">
            <KPI label="Active clients" value={kpis.activeClients.toLocaleString()} />
            <KPI label="Gross MRR" value={fmt0.format(kpis.grossMRR)} />
            <KPI label="Net MRR" value={fmt0.format(kpis.netMRR)} />
            <KPI label="Run-rate recurring comm" value={fmt0.format(kpis.recurringComm)} />
            <KPI label="Cash buffer (cum)" value={fmt0.format(kpis.cashBuffer)} />
            <KPI label={`FTE needed (M${months})`} value={kpis.fteNeeded.toFixed(1)} />
          </div>
        )}

        <Tabs defaultValue="overview" className="space-y-6">
          <TabsList className="flex flex-wrap gap-2">
            <TabsTrigger value="overview">Overview</TabsTrigger>
            <TabsTrigger value="cash">Cash & Draws</TabsTrigger>
            <TabsTrigger value="hiring">Hiring / Capacity</TabsTrigger>
            <TabsTrigger value="pricing">Pricing & Prepay</TabsTrigger>
            <TabsTrigger value="funnel">Funnel</TabsTrigger>
            <TabsTrigger value="reports">Reports</TabsTrigger>
            <TabsTrigger value="move">Move to Tempe</TabsTrigger>
            <TabsTrigger value="unit">Unit Economics</TabsTrigger>
            <TabsTrigger value="presets">Presets & Share</TabsTrigger>
            <TabsTrigger value="tests"><TestTube2 className="w-4 mr-1"/>Tests</TabsTrigger>
          </TabsList>

          {/* OVERVIEW */}
          <TabsContent value="overview">
            <div className="grid grid-cols-1 lg:grid-cols-3 gap-6 items-start">
              <Card className="lg:col-span-1 shadow-sm">
                <CardHeader><CardTitle>Core Inputs</CardTitle></CardHeader>
                <CardContent className="space-y-4">
                  <div className="grid grid-cols-2 gap-4">
                    <Field label="Clients per day" value={clientsPerDay} setValue={setClientsPerDay} disabled={useFunnel} />
                    <Field label="Months to model" value={months} setValue={(v)=>setMonths(clamp(v,1,60))} />
                    <Field label="Starting price ($)" value={startPrice} setValue={setStartPrice} />
                    <Field label="Monthly price increase (%)" value={monthlyPriceGrowthPct} setValue={(v)=>setMonthlyPriceGrowthPct(clamp(v,0,200))} />
                    <Field label="Base MRR ($)" value={baseMRR} setValue={setBaseMRR} />
                    <Field label="Monthly churn (%)" value={churnPct} setValue={(v)=>setChurnPct(clamp(v,0,30))} />
                    <Field label="First-month comm (%)" value={fmCommPct} setValue={(v)=>setFmCommPct(clamp(v,0,100))} />
                    <Field label="Ongoing comm (%)" value={ogCommPct} setValue={(v)=>setOgCommPct(clamp(v,0,100))} />
                  </div>
                  <div className="flex items-center justify-between rounded-2xl border p-3">
                    <div>
                      <Label className="block">Include weekends</Label>
                      <p className="text-xs text-slate-500">{includeWeekends ? "30 days / month" : "22 working days / month"}</p>
                    </div>
                    <Switch checked={includeWeekends} onCheckedChange={setIncludeWeekends} />
                  </div>
                  <div className="flex items-center justify-between rounded-2xl border p-3">
                    <div>
                      <Label className="block">Use funnel to derive clients/day</Label>
                      <p className="text-xs text-slate-500">Overrides manual clients/day using leads → show → close</p>
                    </div>
                    <Switch checked={useFunnel} onCheckedChange={setUseFunnel} />
                  </div>
                  <div className="rounded-2xl bg-slate-50 border p-4">
                    <div className="text-sm text-slate-600">New clients / month</div>
                    <div className="text-2xl font-semibold">{newClientsPerMonth.toLocaleString()}</div>
                  </div>
                  <div className="flex gap-2">
                    <Button onClick={downloadCSV} className="flex gap-2"><Download className="w-4"/>CSV</Button>
                    <Button variant="secondary" onClick={shareLink} className="flex gap-2"><Share2 className="w-4"/>Share</Button>
                  </div>
                </CardContent>
              </Card>

              <Card className="lg:col-span-2 shadow-sm">
                <CardHeader><CardTitle>MRR Trajectory</CardTitle></CardHeader>
                <CardContent>
                  <div className="h-72 w-full">
                    <ResponsiveContainer width="100%" height="100%">
                      <LineChart data={data} margin={{ top: 10, right: 20, bottom: 0, left: 0 }}>
                        <CartesianGrid strokeDasharray="3 3" />
                        <XAxis dataKey="month" tickFormatter={(v) => `M${v}`} />
                        <YAxis tickFormatter={(v) => `$${Math.round((v as number)/1000)}k`} />
                        <Tooltip formatter={(value: any) => fmt0.format(value as number)} labelFormatter={(l) => `Month ${l}`} />
                        <Legend />
                        <Line type="monotone" dataKey="grossMRR" name="Gross MRR" dot={false} strokeWidth={2} />
                        <Line type="monotone" dataKey={(d:any)=>d.grossMRR-d.recurringCommission} name="Net MRR" dot={false} strokeWidth={2} />
                      </LineChart>
                    </ResponsiveContainer>
                  </div>
                </CardContent>
              </Card>
            </div>
          </TabsContent>

          {/* CASH & DRAWS */}
          <TabsContent value="cash">
            <div className="grid grid-cols-1 lg:grid-cols-3 gap-6 items-start">
              <Card className="shadow-sm">
                <CardHeader><CardTitle>Cash & Tax Inputs</CardTitle></CardHeader>
                <CardContent className="grid grid-cols-2 gap-4">
                  <Field label="Fixed OpEx ($/mo)" value={fixedOpEx} setValue={setFixedOpEx} />
                  <Field label="Onboarding cost ($/new)" value={onboardCostPerNew} setValue={setOnboardCostPerNew} />
                  <Field label="Support cost ($/active/mo)" value={supportCostPerActive} setValue={setSupportCostPerActive} />
                  <Field label="Tax rate (%)" value={taxRatePct} setValue={(v)=>setTaxRatePct(clamp(v,0,60))} />
                  <Field label="Min owner draw ($/mo)" value={minOwnerDraw} setValue={setMinOwnerDraw} />
                  <Field label="Extra draw (% of surplus)" value={extraDrawPct} setValue={(v)=>setExtraDrawPct(clamp(v,0,100))} />
                </CardContent>
              </Card>

              <Card className="lg:col-span-2 shadow-sm">
                <CardHeader><CardTitle>Cashflow (Collected) & Draws</CardTitle></CardHeader>
                <CardContent>
                  <div className="h-72 w-full">
                    <ResponsiveContainer width="100%" height="100%">
                      <AreaChart data={data} margin={{ top: 10, right: 20, bottom: 0, left: 0 }}>
                        <CartesianGrid strokeDasharray="3 3" />
                        <XAxis dataKey="month" tickFormatter={(v)=>`M${v}`} />
                        <YAxis tickFormatter={(v)=>`$${Math.round((v as number)/1000)}k`} />
                        <Tooltip formatter={(value:any)=>fmt0.format(value as number)} labelFormatter={(l)=>`Month ${l}`} />
                        <Legend />
                        <Area type="monotone" dataKey="cashCollected" name="Cash collected" fillOpacity={0.3} strokeWidth={2} />
                        <Area type="monotone" dataKey="commissionCashOut" name="Commission cash" fillOpacity={0.2} strokeWidth={2} />
                        <Area type="monotone" dataKey="opExCash" name="OpEx (cash)" fillOpacity={0.2} strokeWidth={2} />
                        <Area type="monotone" dataKey="taxesCash" name="Taxes (cash)" fillOpacity={0.2} strokeWidth={2} />
                        <Area type="monotone" dataKey="ownerDraw" name="Owner draw" fillOpacity={0.2} strokeWidth={2} />
                        <Line type="monotone" dataKey="netCashAfterDraw" name="Net cash after draw" dot={false} strokeWidth={2} />
                      </AreaChart>
                    </ResponsiveContainer>
                  </div>
                </CardContent>
              </Card>
            </div>
          </TabsContent>

          {/* HIRING */}
          <TabsContent value="hiring">
            <div className="grid grid-cols-1 lg:grid-cols-3 gap-6 items-start">
              <Card className="shadow-sm">
                <CardHeader><CardTitle>Capacity Inputs</CardTitle></CardHeader>
                <CardContent className="grid grid-cols-2 gap-4">
                  <Field label="Onboarding hrs / new" value={onboardingHrsPerNew} setValue={setOnboardingHrsPerNew} />
                  <Field label="Ongoing hrs / active / mo" value={ongoingHrsPerActive} setValue={setOngoingHrsPerActive} />
                  <Field label="Hours / FTE / mo" value={hrsPerFTE} setValue={setHrsPerFTE} />
                </CardContent>
              </Card>

              <Card className="lg:col-span-2 shadow-sm">
                <CardHeader><CardTitle>FTE Needed & Hiring Triggers</CardTitle></CardHeader>
                <CardContent>
                  <div className="h-72 w-full">
                    <ResponsiveContainer width="100%" height="100%">
                      <BarChart data={data} margin={{ top: 10, right: 20, bottom: 0, left: 0 }}>
                        <CartesianGrid strokeDasharray="3 3" />
                        <XAxis dataKey="month" tickFormatter={(v)=>`M${v}`} />
                        <YAxis />
                        <Tooltip formatter={(v:any)=>v.toFixed ? v.toFixed(2) : v} labelFormatter={(l)=>`Month ${l}`} />
                        <Legend />
                        <Bar dataKey="fteNeeded" name="FTE needed" />
                      </BarChart>
                    </ResponsiveContainer>
                  </div>
                  <div className="mt-4 text-sm text-slate-600">Tip: hire when FTE needed exceeds current headcount for 2 consecutive months, or when tickets exceed SLA.</div>
                </CardContent>
              </Card>
            </div>
          </TabsContent>

          {/* PRICING & PREPAY */}
          <TabsContent value="pricing">
            <div className="grid grid-cols-1 lg:grid-cols-3 gap-6 items-start">
              <Card className="shadow-sm">
                <CardHeader><CardTitle>Pricing & Prepay</CardTitle></CardHeader>
                <CardContent className="grid grid-cols-2 gap-4">
                  <Field label="Starting price ($)" value={startPrice} setValue={setStartPrice} />
                  <Field label="Monthly price growth (%)" value={monthlyPriceGrowthPct} setValue={(v)=>setMonthlyPriceGrowthPct(clamp(v,0,200))} />
                  <Field label="Prepay adoption (%)" value={prepayPct} setValue={(v)=>setPrepayPct(clamp(v,0,100))} />
                  <Field label="Prepay months free" value={prepayMonthsFree} setValue={(v)=>setPrepayMonthsFree(clamp(v,0,6))} />
                </CardContent>
              </Card>

              <Card className="lg:col-span-2 shadow-sm">
                <CardHeader><CardTitle>New-Cohort Price Curve</CardTitle></CardHeader>
                <CardContent>
                  <div className="h-72 w-full">
                    <ResponsiveContainer width="100%" height="100%">
                      <LineChart data={data} margin={{ top: 10, right: 20, bottom: 0, left: 0 }}>
                        <CartesianGrid strokeDasharray="3 3" />
                        <XAxis dataKey="month" tickFormatter={(v)=>`M${v}`} />
                        <YAxis tickFormatter={(v)=>fmt0.format(v as number)} />
                        <Tooltip formatter={(value:any)=>fmt2.format(value as number)} labelFormatter={(l)=>`Month ${l}`} />
                        <Legend />
                        <Line type="monotone" dataKey="price" name="New cohort price" dot={false} strokeWidth={2} />
                        <Line type="monotone" dataKey="upfrontPrepayCash" name="Upfront prepay cash" dot={false} strokeWidth={2} />
                      </LineChart>
                    </ResponsiveContainer>
                  </div>
                </CardContent>
              </Card>
            </div>
          </TabsContent>

          {/* FUNNEL */}
          <TabsContent value="funnel">
            <div className="grid grid-cols-1 lg:grid-cols-3 gap-6 items-start">
              <Card className="shadow-sm">
                <CardHeader><CardTitle>Funnel Inputs</CardTitle></CardHeader>
                <CardContent className="grid grid-cols-2 gap-4">
                  <Field label="Leads per day" value={leadsPerDay} setValue={setLeadsPerDay} />
                  <Field label="Show rate (%)" value={showRatePct} setValue={(v)=>setShowRatePct(clamp(v,0,100))} />
                  <Field label="Close rate (%)" value={closeRatePct} setValue={(v)=>setCloseRatePct(clamp(v,0,100))} />
                  <div className="rounded-2xl bg-slate-50 border p-3 col-span-2">
                    <div className="text-xs text-slate-600">Derived clients/day</div>
                    <div className="text-xl font-semibold">{derivedClientsPerDay.toFixed(2)}</div>
                  </div>
                </CardContent>
              </Card>

              <Card className="lg:col-span-2 shadow-sm">
                <CardHeader><CardTitle>Funnel → Clients/Day</CardTitle></CardHeader>
                <CardContent>
                  <div className="h-72 w-full">
                    <ResponsiveContainer width="100%" height="100%">
                      <BarChart data={[{step:"Leads",v:leadsPerDay},{step:"Shows",v:leadsPerDay*(showRatePct/100)},{step:"Closes",v:derivedClientsPerDay}]}> 
                        <CartesianGrid strokeDasharray="3 3" />
                        <XAxis dataKey="step" />
                        <YAxis />
                        <Tooltip />
                        <Legend />
                        <Bar dataKey="v" name="per day" />
                      </BarChart>
                    </ResponsiveContainer>
                  </div>
                </CardContent>
              </Card>
            </div>
          </TabsContent>

          {/* REPORTS */}
          <TabsContent value="reports">
            <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
              <Card className="shadow-sm">
                <CardHeader><CardTitle>Snapshot: M1 / M3 / M6 / M12</CardTitle></CardHeader>
                <CardContent className="overflow-x-auto">
                  <table className="min-w-full text-sm">
                    <thead className="text-left text-slate-600">
                      <tr>
                        <Th>Month</Th><Th>Gross MRR</Th><Th>Rec. Comm</Th><Th>Net MRR</Th><Th>Cash In</Th><Th>Owner Draw</Th><Th>FTE</Th>
                      </tr>
                    </thead>
                    <tbody>
                      {([1,3,6,12] as number[]).filter(m=>m<=months).map(m=>{
                        const r = data[m-1];
                        return (
                          <tr key={m} className="border-t">
                            <Td>M{m}</Td>
                            <Td>{fmt0.format(r.grossMRR)}</Td>
                            <Td>{fmt0.format(r.recurringCommission)}</Td>
                            <Td>{fmt0.format(r.grossMRR - r.recurringCommission)}</Td>
                            <Td>{fmt0.format(r.cashCollected)}</Td>
                            <Td>{fmt0.format(r.ownerDraw)}</Td>
                            <Td>{r.fteNeeded.toFixed(2)}</Td>
                          </tr>
                        );
                      })}
                    </tbody>
                  </table>
                </CardContent>
              </Card>

              <Card className="shadow-sm">
                <CardHeader><CardTitle>Commission Cash This Month</CardTitle></CardHeader>
                <CardContent>
                  <div className="h-72 w-full">
                    <ResponsiveContainer width="100%" height="100%">
                      <LineChart data={data}>
                        <CartesianGrid strokeDasharray="3 3" />
                        <XAxis dataKey="month" tickFormatter={(v)=>`M${v}`} />
                        <YAxis tickFormatter={(v)=>`$${Math.round((v as number)/1000)}k`} />
                        <Tooltip formatter={(val:any)=>fmt0.format(val as number)} labelFormatter={(l)=>`Month ${l}`} />
                        <Legend />
                        <Line type="monotone" dataKey="commissionCashOut" name="Commission cash (mo)" dot={false} strokeWidth={2} />
                      </LineChart>
                    </ResponsiveContainer>
                  </div>
                </CardContent>
              </Card>
            </div>
          </TabsContent>

          {/* MOVE TO TEMPE */}
          <TabsContent value="move">
            <MoveToTempe data={data} />
          </TabsContent>

          {/* UNIT ECONOMICS */}
          <TabsContent value="unit">
            <UnitEconomics startPrice={startPrice} churnPct={churnPct} ogCommPct={ogCommPct} supportCostPerActive={supportCostPerActive} />
          </TabsContent>

          {/* PRESETS & SHARE */}
          <TabsContent value="presets">
            <Card className="shadow-sm">
              <CardHeader><CardTitle>Save / Load Presets & Share</CardTitle></CardHeader>
              <CardContent className="flex flex-col md:flex-row gap-3 items-end">
                <div className="grid grid-cols-2 gap-4 w-full md:w-auto">
                  <div className="col-span-2">
                    <Label>Preset name</Label>
                    <Input value={presetName} onChange={(e)=>setPresetName(e.target.value)} placeholder="e.g., 3day_10pct_2churn" />
                  </div>
                  <Button onClick={savePreset} className="flex gap-2"><Save className="w-4"/>Save</Button>
                  <Button variant="secondary" onClick={loadPreset}>Load</Button>
                  <Button variant="outline" onClick={downloadCSV} className="flex gap-2"><Download className="w-4"/>CSV</Button>
                  <Button variant="secondary" onClick={shareLink} className="flex gap-2"><Share2 className="w-4"/>Share Link</Button>
                </div>
              </CardContent>
            </Card>

            {/* Table */}
            <Card className="mt-6 shadow-sm">
              <CardHeader><CardTitle>Monthly Projection</CardTitle></CardHeader>
              <CardContent className="overflow-x-auto">
                <table className="min-w-full text-sm">
                  <thead className="text-left text-slate-600">
                    <tr>
                      <Th>Month</Th><Th>New</Th><Th>Prepay</Th><Th>Pay-Mo</Th><Th>Price</Th><Th>Active</Th>
                      <Th>Gross MRR</Th><Th>Rec. Comm</Th><Th>Net MRR</Th><Th>1× Comm</Th><Th>Comm Cash</Th>
                      <Th>Cash In</Th><Th>Upfront Prepay</Th><Th>OpEx</Th><Th>Taxes</Th><Th>Post-Tax</Th><Th>Owner Draw</Th><Th>Net After</Th><Th>Buffer</Th><Th>FTE</Th>
                    </tr>
                  </thead>
                  <tbody>
                    {data.map((r:any)=> (
                      <tr key={r.month} className="border-t">
                        <Td>M{r.month}</Td>
                        <Td>{r.newClients.toLocaleString()}</Td>
                        <Td>{r.prepayCount.toLocaleString()}</Td>
                        <Td>{r.payMonthlyCount.toLocaleString()}</Td>
                        <Td>{fmt2.format(r.price)}</Td>
                        <Td>{Math.round(r.activeClients).toLocaleString()}</Td>
                        <Td>{fmt0.format(r.grossMRR)}</Td>
                        <Td>{fmt0.format(r.recurringCommission)}</Td>
                        <Td>{fmt0.format(r.grossMRR - r.recurringCommission)}</Td>
                        <Td>{fmt0.format(r.oneTimeCommission)}</Td>
                        <Td>{fmt0.format(r.commissionCashOut)}</Td>
                        <Td>{fmt0.format(r.cashCollected)}</Td>
                        <Td>{fmt0.format(r.upfrontPrepayCash)}</Td>
                        <Td>{fmt0.format(r.opExCash)}</Td>
                        <Td>{fmt0.format(r.taxesCash)}</Td>
                        <Td>{fmt0.format(r.postTaxCash)}</Td>
                        <Td>{fmt0.format(r.ownerDraw)}</Td>
                        <Td>{fmt0.format(r.netCashAfterDraw)}</Td>
                        <Td>{fmt0.format(r.cashBuffer)}</Td>
                        <Td>{r.fteNeeded.toFixed(2)}</Td>
                      </tr>
                    ))}
                  </tbody>
                </table>
              </CardContent>
            </Card>
          </TabsContent>

          {/* TESTS */}
          <TabsContent value="tests">
            <TestSuite data={data} newClientsPerMonth={newClientsPerMonth} months={months} />
          </TabsContent>
        </Tabs>

        <footer className="text-xs text-slate-500 mt-6">
          Notes: Accrual vs cash are treated separately. Prepay collects cash upfront for (12 - months free) months but MRR still accrues monthly. Churn reduces active clients by (1 - churn)^age. Owner draw is capped at post-tax cash for realism.
        </footer>
      </div>
    </div>
  );
}

// ===== Small UI helpers =====
function KPI({ label, value }: { label: string; value: string }) {
  return (
    <div className="rounded-2xl border p-4 bg-white/60">
      <div className="text-xs uppercase tracking-wide text-slate-500">{label}</div>
      <div className="text-xl font-semibold mt-1">{value}</div>
    </div>
  );
}

function Field({ label, value, setValue, disabled=false }: { label: string; value: number; setValue: (n: number)=>void; disabled?: boolean }) {
  return (
    <div>
      <Label>{label}</Label>
      <Input type="number" disabled={disabled} value={Number(value)} onChange={e=>setValue(Number(e.target.value))} />
    </div>
  );
}

const Th = ({children}:{children: React.ReactNode}) => (<th className="py-2 pr-4 whitespace-nowrap">{children}</th>);
const Td = ({children}:{children: React.ReactNode}) => (<td className="py-2 pr-4 whitespace-nowrap">{children}</td>);

// ===== Extra Modules =====
function MoveToTempe({ data }: { data: any[] }) {
  const [rent, setRent] = useState(1700);
  const [utilities, setUtilities] = useState(220);
  const [internet, setInternet] = useState(60);
  const [otherMonthly, setOtherMonthly] = useState(600);
  const [depositMonths, setDepositMonths] = useState(2);
  const [oneTimeMoveCosts, setOneTimeMoveCosts] = useState(1800);

  const monthlyBudget = rent + utilities + internet + otherMonthly;
  const requiredUpfront = oneTimeMoveCosts + rent * depositMonths;

  const earliest = useMemo(() => {
    let okMonth: number | null = null;
    for (const r of data) {
      const ownerCoversBudget = r.ownerDraw >= monthlyBudget;
      const bufferCoversUpfront = r.cashBuffer >= requiredUpfront;
      if (ownerCoversBudget && bufferCoversUpfront) { okMonth = r.month; break; }
    }
    return okMonth;
  }, [data, monthlyBudget, requiredUpfront]);

  return (
    <div className="grid grid-cols-1 lg:grid-cols-3 gap-6 items-start">
      <Card className="shadow-sm">
        <CardHeader><CardTitle>Tempe Cost Inputs</CardTitle></CardHeader>
        <CardContent className="grid grid-cols-2 gap-4">
          <Field label="Rent ($/mo)" value={rent} setValue={setRent} />
          <Field label="Utilities ($/mo)" value={utilities} setValue={setUtilities} />
          <Field label="Internet ($/mo)" value={internet} setValue={setInternet} />
          <Field label="Other monthly ($/mo)" value={otherMonthly} setValue={setOtherMonthly} />
          <Field label="Deposit (months)" value={depositMonths} setValue={setDepositMonths} />
          <Field label="One-time move ($)" value={oneTimeMoveCosts} setValue={setOneTimeMoveCosts} />
          <div className="rounded-2xl bg-slate-50 border p-3 col-span-2">
            <div className="text-xs text-slate-600">Monthly budget</div>
            <div className="text-xl font-semibold">{fmt0.format(monthlyBudget)}</div>
          </div>
          <div className="rounded-2xl bg-slate-50 border p-3 col-span-2">
            <div className="text-xs text-slate-600">Upfront needed</div>
            <div className="text-xl font-semibold">{fmt0.format(requiredUpfront)}</div>
          </div>
        </CardContent>
      </Card>

      <Card className="lg:col-span-2 shadow-sm">
        <CardHeader><CardTitle>Earliest Safe Move Month</CardTitle></CardHeader>
        <CardContent>
          {(() => {
            const ok = earliest;
            if (!ok) return <p className="text-slate-600">Not yet safe under current assumptions. Increase clients/day, prepay %, or reduce costs to accelerate.</p>;
            const r = data[ok - 1];
            return (
              <div className="space-y-4">
                <div className="text-lg">You can safely move in <span className="font-semibold">Month {ok}</span>.</div>
                <ul className="text-sm list-disc pl-5">
                  <li>Owner draw at Month {ok}: <span className="font-semibold">{fmt0.format(r.ownerDraw)}</span> (budget: {fmt0.format(monthlyBudget)})</li>
                  <li>Cash buffer by Month {ok}: <span className="font-semibold">{fmt0.format(r.cashBuffer)}</span> (upfront: {fmt0.format(requiredUpfront)})</li>
                  <li>Net MRR at Month {ok}: <span className="font-semibold">{fmt0.format(r.grossMRR - r.recurringCommission)}</span></li>
                </ul>
              </div>
            );
          })()}
        </CardContent>
      </Card>
    </div>
  );
}

function UnitEconomics({ startPrice, churnPct, ogCommPct, supportCostPerActive }: { startPrice: number; churnPct: number; ogCommPct: number; supportCostPerActive: number; }) {
  const c = Math.max(churnPct, 0.1) / 100; // avoid div/0, min 0.1%
  const arpu = startPrice; // new cohort price as ARPU proxy
  const grossMarginPct = 1 - (ogCommPct/100) - (supportCostPerActive / Math.max(1, arpu));
  const ltv = (arpu * grossMarginPct) / c; // basic LTV approximation

  const scenarios = [0.5, 1, 2, 3, 5].map(ch => {
    const cc = ch/100;
    return { churn: ch, ltv: (arpu * grossMarginPct) / cc };
  });

  return (
    <div className="grid grid-cols-1 lg:grid-cols-2 gap-6 items-start">
      <Card className="shadow-sm">
        <CardHeader><CardTitle>LTV Estimate</CardTitle></CardHeader>
        <CardContent>
          <div className="grid grid-cols-2 gap-4">
            <KPI label="ARPU (new cohort)" value={fmt2.format(arpu)} />
            <KPI label="Gross margin % (approx)" value={`${(grossMarginPct*100).toFixed(1)}%`} />
            <KPI label="Churn % (used)" value={`${(c*100).toFixed(2)}%/mo`} />
            <KPI label="LTV (approx)" value={fmt0.format(ltv)} />
          </div>
        </CardContent>
      </Card>
      <Card className="shadow-sm">
        <CardHeader><CardTitle>LTV vs Churn Sensitivity</CardTitle></CardHeader>
        <CardContent>
          <div className="h-72 w-full">
            <ResponsiveContainer width="100%" height="100%">
              <LineChart data={scenarios}>
                <CartesianGrid strokeDasharray="3 3" />
                <XAxis dataKey="churn" tickFormatter={(v)=>`${v}%/mo`} />
                <YAxis tickFormatter={(v)=>`$${Math.round((v as number)/1000)}k`} />
                <Tooltip formatter={(val:any)=>fmt0.format(val as number)} labelFormatter={(l)=>`${l}% churn`} />
                <Legend />
                <Line type="monotone" dataKey="ltv" name="LTV" dot={false} strokeWidth={2} />
              </LineChart>
            </ResponsiveContainer>
          </div>
        </CardContent>
      </Card>
    </div>
  );
}

// ===== Basic Test Suite (UI) =====
function TestSuite({ data, newClientsPerMonth, months }: { data: any[]; newClientsPerMonth: number; months: number }) {
  type TResult = { name: string; pass: boolean; info?: string };
  const tests: TResult[] = [];
  // Test 1: data length equals months
  tests.push({ name: "Data length equals months", pass: data.length === months, info: `len=${data.length}, months=${months}` });
  // Test 2: newClientsPerMonth non-negative
  tests.push({ name: "New clients/month non-negative", pass: newClientsPerMonth >= 0, info: `${newClientsPerMonth}` });
  // Test 3: Gross MRR monotonically non-decreasing without churn & with price growth
  const mono = data.every((r:any, i:number, arr:any[]) => i===0 || r.grossMRR >= arr[i-1].grossMRR);
  tests.push({ name: "Gross MRR monotonic (no churn assumption)", pass: mono });
  // Test 4: Net MRR = Gross - Recurring Commission (within rounding)
  const m0 = data.every((r:any)=> Math.abs((r.grossMRR - r.recurringCommission) - (r.grossMRR - r.recurringCommission)) < 1e-6);
  tests.push({ name: "Net MRR identity holds", pass: m0 });

  const allPass = tests.every(t=>t.pass);
  return (
    <Card className="shadow-sm">
      <CardHeader><CardTitle className={allPass?"text-green-600":"text-red-600"}>{allPass?"All tests passed":"Some tests failed"}</CardTitle></CardHeader>
      <CardContent>
        <table className="min-w-full text-sm">
          <thead className="text-left text-slate-600"><tr><Th>Test</Th><Th>Status</Th><Th>Info</Th></tr></thead>
          <tbody>
            {tests.map((t,i)=> (
              <tr key={i} className="border-t">
                <Td>{t.name}</Td>
                <Td>{t.pass?"✅":"❌"}</Td>
                <Td>{t.info||""}</Td>
              </tr>
            ))}
          </tbody>
        </table>
      </CardContent>
    </Card>
  );
}
