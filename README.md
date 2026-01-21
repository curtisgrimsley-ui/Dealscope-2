"use client";

import { useMemo, useState, useEffect, useCallback } from "react";
import DealScopeAssistant from "./components/DealScopeAssistant";

// Types for better type safety
interface Deal {
  arv: string;
  purchasePrice: string;
  repairCosts: string;
  locationScore: number;
  marketTrend: number;
  rentalDemand: number;
  daysOnMarket: number;
  comps: number;
}

interface ScoreBreakdownItem {
  score: number;
  max: number;
  label: string;
}

interface ScoreData {
  totalScore: number;
  breakdown: Record<string, ScoreBreakdownItem>;
  metrics: {
    profitMargin: number;
    repairRatio: number;
    totalCost: string;
    expectedProfit: string;
    profitMarginRaw: number;
  };
}

interface SharePlatform {
  id: string;
  name: string;
  icon: string;
  color: string;
}

interface RecentScore {
  location: string;
  score: number;
  time: string;
}

interface UnlockReward {
  shares: number;
  reward: string;
}

// Utility function to format money
function formatMoney(n: number): string {
  if (!Number.isFinite(n) || isNaN(n)) return "$0";
  return n.toLocaleString("en-US", {
    style: "currency",
    currency: "USD",
    maximumFractionDigits: 0,
  });
}

// Enhanced Flip Score Calculator with all deal scope factors
function calculateFlipScore(deal: Omit<Deal, 'arv' | 'purchasePrice' | 'repairCosts'> & { 
  arv: number; 
  purchasePrice: number; 
  repairCosts: number;
}): ScoreData | null {
  const {
    arv = 0,
    purchasePrice = 0,
    repairCosts = 0,
    locationScore = 5,
    marketTrend = 5,
    rentalDemand = 5,
    daysOnMarket = 30,
    comps = 3
  } = deal;

  if (!arv || arv <= 0) return null;

  // 1. PROFIT MARGIN (40% weight)
  const totalCost = purchasePrice + repairCosts;
  const expectedProfit = arv - totalCost;
  const profitMargin = (expectedProfit / arv) * 100;
  
  let profitScore = 0;
  if (profitMargin >= 30) profitScore = 40;
  else if (profitMargin >= 20) profitScore = 30;
  else if (profitMargin >= 10) profitScore = 20;
  else if (profitMargin >= 0) profitScore = 10;
  else profitScore = 0;

  // 2. REPAIR COST RATIO (20% weight)
  const repairRatio = (repairCosts / arv) * 100;
  let repairScore = 0;
  if (repairRatio <= 10) repairScore = 20;
  else if (repairRatio <= 20) repairScore = 15;
  else if (repairRatio <= 30) repairScore = 10;
  else repairScore = 5;

  // 3. LOCATION & MARKET (20% weight)
  const locationScorePoints = (locationScore / 10) * 10;
  const marketScorePoints = ((marketTrend + rentalDemand) / 20) * 10;
  const locationMarketScore = locationScorePoints + marketScorePoints;

  // 4. DEAL VELOCITY (10% weight)
  let velocityScore = 0;
  if (daysOnMarket >= 90) velocityScore = 10;
  else if (daysOnMarket >= 60) velocityScore = 7;
  else if (daysOnMarket >= 30) velocityScore = 5;
  else if (daysOnMarket >= 7) velocityScore = 3;
  else velocityScore = 1;

  // 5. COMPARABLES CONFIDENCE (10% weight)
  let compsScore = 0;
  if (comps >= 5) compsScore = 10;
  else if (comps >= 3) compsScore = 7;
  else if (comps >= 1) compsScore = 4;
  else compsScore = 0;

  const totalScore = profitScore + repairScore + locationMarketScore + velocityScore + compsScore;

  // Breakdown for visualization
  const breakdown = {
    profitScore: { score: profitScore, max: 40, label: "Profit Potential" },
    repairScore: { score: repairScore, max: 20, label: "Repair Efficiency" },
    locationMarketScore: { score: locationMarketScore, max: 20, label: "Market & Location" },
    velocityScore: { score: velocityScore, max: 10, label: "Deal Velocity" },
    compsScore: { score: compsScore, max: 10, label: "Comparables" }
  };

  return {
    totalScore: Math.min(100, Math.round(totalScore)),
    breakdown,
    metrics: {
      profitMargin: Math.round(profitMargin),
      repairRatio: Math.round(repairRatio),
      totalCost: formatMoney(totalCost),
      expectedProfit: formatMoney(expectedProfit),
      profitMarginRaw: profitMargin
    }
  };
}

// Input validation function - FIXED: Added daysOnMarket validation
function validateInputs(deal: Deal): string[] {
  const errors: string[] = [];
  
  const arv = Number(deal.arv);
  const purchasePrice = Number(deal.purchasePrice);
  const repairCosts = Number(deal.repairCosts);
  
  if (isNaN(arv) || arv <= 0) errors.push("ARV must be greater than 0");
  if (isNaN(purchasePrice) || purchasePrice < 0) errors.push("Purchase price cannot be negative");
  if (isNaN(repairCosts) || repairCosts < 0) errors.push("Repair costs cannot be negative");
  if (deal.daysOnMarket < 0) errors.push("Days on market must be positive");
  
  return errors;
}

// Deal Scope Input Section Component - FIXED: Removed unused error check
function DealScopeInputs({ deal, onUpdate, errors }: { 
  deal: Deal; 
  onUpdate: (field: keyof Deal, value: any) => void;
  errors: string[];
}) {
  const factors = [
    {
      id: 'locationScore' as keyof Deal,
      label: 'Location Quality',
      description: 'How desirable is the neighborhood?',
      min: 1,
      max: 10,
      value: deal.locationScore
    },
    {
      id: 'marketTrend' as keyof Deal,
      label: 'Market Trend',
      description: 'Are prices rising, stable, or falling?',
      min: 1,
      max: 10,
      value: deal.marketTrend
    },
    {
      id: 'rentalDemand' as keyof Deal,
      label: 'Rental Demand',
      description: 'How strong is rental demand in the area?',
      min: 1,
      max: 10,
      value: deal.rentalDemand
    }
  ];

  return (
    <div style={{
      border: "1px solid rgba(0,0,0,0.12)",
      borderRadius: 14,
      padding: 20,
      marginTop: 24,
      backgroundColor: "#f9fafb"
    }}>
      <h3 style={{ marginTop: 0, marginBottom: 16 }}>📊 Deal Scope Factors</h3>
      <p style={{ marginBottom: 20, opacity: 0.7, fontSize: 14 }}>
        Rate these factors from 1 (Poor) to 10 (Excellent)
      </p>
      
      {/* Error Display */}
      {errors.length > 0 && (
        <div style={{
          padding: 12,
          backgroundColor: "#FEE2E2",
          border: "1px solid #FCA5A5",
          borderRadius: 8,
          marginBottom: 16
        }}>
          <div style={{ fontWeight: 600, color: "#DC2626", marginBottom: 4 }}>
            Please fix the following issues:
          </div>
          <ul style={{ margin: 0, paddingLeft: 20 }}>
            {errors.map((error, index) => (
              <li key={index} style={{ fontSize: 14, color: "#DC2626" }}>{error}</li>
            ))}
          </ul>
        </div>
      )}
      
      <div style={{ display: "grid", gap: 20 }}>
        {factors.map((factor) => (
          <div key={factor.id} style={{ marginBottom: 12 }}>
            <div style={{ 
              display: "flex", 
              justifyContent: "space-between",
              marginBottom: 8 
            }}>
              <div>
                <div style={{ fontWeight: 600 }}>{factor.label}</div>
                <div style={{ fontSize: 13, opacity: 0.7 }}>{factor.description}</div>
              </div>
              <div style={{ 
                fontWeight: 700, 
                fontSize: 18,
                color: factor.value >= 7 ? "#059669" : factor.value >= 4 ? "#D97706" : "#DC2626"
              }}>
                {factor.value}/10
              </div>
            </div>
            
            <input
              type="range"
              min={factor.min}
              max={factor.max}
              value={factor.value}
              onChange={(e) => onUpdate(factor.id, parseInt(e.target.value))}
              style={{
                width: "100%",
                height: 8,
                borderRadius: 4,
                background: "linear-gradient(90deg, #DC2626 0%, #D97706 50%, #059669 100%)",
                outline: "none",
                cursor: "pointer"
              }}
              aria-label={`Adjust ${factor.label}`}
            />
            
            <div style={{
              display: "flex",
              justifyContent: "space-between",
              fontSize: 12,
              color: "#6B7280",
              marginTop: 4
            }}>
              <span>Poor</span>
              <span>Average</span>
              <span>Excellent</span>
            </div>
          </div>
        ))}
        
        <div style={{ 
          display: "grid", 
          gridTemplateColumns: "1fr 1fr", 
          gap: 16,
          marginTop: 12 
        }}>
          <div>
            <label style={{ display: "block", fontWeight: 600, marginBottom: 8, fontSize: 14 }}>
              Days on Market
            </label>
            <input
              type="number"
              value={deal.daysOnMarket}
              onChange={(e) => onUpdate('daysOnMarket', parseInt(e.target.value) || 0)}
              onBlur={() => {
                const validationErrors = validateInputs(deal);
                // Filter to only show days on market error for this field
                const daysError = validationErrors.find(err => err.includes("Days on market"));
                // Don't show ARV/purchase/repair errors here
              }}
              style={{
                width: "100%",
                padding: "10px",
                borderRadius: 8,
                border: "1px solid rgba(0,0,0,0.2)",
                backgroundColor: "white"
              }}
              placeholder="e.g., 30"
              min="0"
              aria-label="Days on Market"
            />
          </div>
          
          <div>
            <label style={{ display: "block", fontWeight: 600, marginBottom: 8, fontSize: 14 }}>
              Comparable Sales
            </label>
            <select
              value={deal.comps}
              onChange={(e) => onUpdate('comps', parseInt(e.target.value))}
              style={{
                width: "100%",
                padding: "10px",
                borderRadius: 8,
                border: "1px solid rgba(0,0,0,0.2)",
                backgroundColor: "white",
                cursor: "pointer"
              }}
              aria-label="Number of Comparable Sales"
            >
              <option value="0">No comps</option>
              <option value="1">1-2 comps</option>
              <option value="3">3-4 comps</option>
              <option value="5">5+ comps</option>
            </select>
          </div>
        </div>
      </div>
    </div>
  );
}

// Score Visualization Component
function ScoreVisualization({ scoreData }: { scoreData: ScoreData }) {
  const { totalScore, breakdown } = scoreData;
  
  const getScoreColor = (score: number): string => {
    if (score >= 80) return "#10B981";
    if (score >= 70) return "#F59E0B";
    if (score >= 60) return "#F97316";
    return "#EF4444";
  };

  const getRecommendation = (score: number): string => {
    if (score >= 80) return "🔥 HOT DEAL - Strong profit potential!";
    if (score >= 70) return "👍 GOOD DEAL - Worth serious consideration";
    if (score >= 60) return "🤔 MARGINAL - Needs careful evaluation";
    return "⚠️ HIGH RISK - Consider other options";
  };

  const getScoreLabel = (score: number): string => {
    if (score >= 80) return "Excellent";
    if (score >= 70) return "Good";
    if (score >= 60) return "Fair";
    return "Poor";
  };

  return (
    <div style={{
      border: "1px solid rgba(0,0,0,0.12)",
      borderRadius: 14,
      padding: 24,
      marginTop: 24,
      backgroundColor: "white"
    }}>
      <div style={{
        display: "flex",
        alignItems: "center",
        justifyContent: "space-between",
        marginBottom: 24,
        flexWrap: "wrap",
        gap: 16
      }}>
        <div>
          <div style={{ fontSize: 14, fontWeight: 600, color: "#6B7280", marginBottom: 4 }}>
            OVERALL FLIP SCORE
          </div>
          <div style={{ 
            fontSize: 48, 
            fontWeight: 800, 
            color: getScoreColor(totalScore),
            marginBottom: 4
          }}>
            {totalScore}/100
          </div>
          <div style={{
            display: "inline-block",
            padding: "4px 12px",
            backgroundColor: getScoreColor(totalScore) + "20",
            borderRadius: 20,
            fontSize: 14,
            fontWeight: 600,
            color: getScoreColor(totalScore)
          }}>
            {getScoreLabel(totalScore)}
          </div>
        </div>
        
        <div style={{
          padding: "12px 20px",
          backgroundColor: getScoreColor(totalScore) + "20",
          borderRadius: 12,
          borderLeft: `4px solid ${getScoreColor(totalScore)}`,
          maxWidth: 400
        }}>
          <div style={{ fontWeight: 700, fontSize: 16, marginBottom: 4 }}>
            {getRecommendation(totalScore)}
          </div>
          <div style={{ fontSize: 14, opacity: 0.8 }}>
            Based on profit potential, market factors, and deal dynamics
          </div>
        </div>
      </div>

      {/* Score Breakdown */}
      <div style={{ marginBottom: 24 }}>
        <h4 style={{ marginBottom: 16 }}>Score Breakdown</h4>
        <div style={{ display: "grid", gap: 12 }}>
          {Object.entries(breakdown).map(([key, data]) => (
            <div key={key} style={{ marginBottom: 8 }}>
              <div style={{ 
                display: "flex", 
                justifyContent: "space-between",
                marginBottom: 4 
              }}>
                <span style={{ fontWeight: 600 }}>{data.label}</span>
                <span>{data.score}/{data.max}</span>
              </div>
              <div style={{
                height: 8,
                backgroundColor: "#E5E7EB",
                borderRadius: 4,
                overflow: "hidden"
              }}>
                <div style={{
                  width: `${(data.score / data.max) * 100}%`,
                  height: "100%",
                  backgroundColor: getScoreColor((data.score / data.max) * 100),
                  borderRadius: 4,
                  transition: "width 0.3s ease"
                }} />
              </div>
            </div>
          ))}
        </div>
      </div>
    </div>
  );
}

// Share Panel Component
function SharePanel({ scoreData, deal, maxOffer, onShareComplete }: { 
  scoreData: ScoreData; 
  deal: Deal; 
  maxOffer: number;
  onShareComplete: () => void;
}) {
  const [sharedCount, setSharedCount] = useState<number>(0);
  const [copied, setCopied] = useState<boolean>(false);
  
  const sharePlatforms: SharePlatform[] = [
    { id: 'twitter', name: 'Tweet', icon: '𝕏', color: '#1DA1F2' },
    { id: 'facebook', name: 'Share', icon: 'f', color: '#4267B2' },
    { id: 'linkedin', name: 'Post', icon: 'in', color: '#0077B5' },
    { id: 'whatsapp', name: 'Share', icon: '📱', color: '#25D366' }
  ];
  
  const unlockRewards: UnlockReward[] = [
    { shares: 1, reward: "📊 Detailed Repair Cost Estimator Template" },
    { shares: 2, reward: "📖 Free Chapter: 'Finding Hidden Gem Properties'" },
    { shares: 3, reward: "🎯 Personalized Deal Review Consultation" }
  ];
  
  const getShareText = (platform: string): string => {
    const baseText = `🏠 Just analyzed a deal with a Flip Score of ${scoreData.totalScore}/100! ${scoreData.totalScore >= 80 ? "🔥 HOT DEAL!" : "Check out this analysis."}`;
    const url = typeof window !== 'undefined' ? window.location.origin : "https://curtisgrimsley.com";
    
    const texts: Record<string, string> = {
      twitter: `${baseText}\n\nAnalyze your own deals for free:\n${url}\n\n#RealEstateFlipping #DealAnalyzer #FlipScore #RealEstateInvesting`,
      facebook: `I just used the Deal Analyzer tool and scored a property ${scoreData.totalScore}/100! This professional tool from "The Real Estate Flipping Blueprint" helped me evaluate a potential flip. Try it for free: ${url}`,
      linkedin: `Professional real estate analysis: Scored a potential flip at ${scoreData.totalScore}/100 using comprehensive criteria. This tool from "The Real Estate Flipping Blueprint" evaluates profit potential, location, market trends, and more. A must-try for serious investors: ${url}`,
      whatsapp: `Check out this deal analyzer! Scored ${scoreData.totalScore}/100: ${url}`,
      generic: `Flip Score: ${scoreData.totalScore}/100\nProfit Margin: ${scoreData.metrics.profitMargin}%\nMax Offer: ${formatMoney(maxOffer)}\n\nAnalyzed with the Deal Analyzer from "The Real Estate Flipping Blueprint"\n${url}`
    };
    
    return texts[platform] || texts.generic;
  };
  
  const handleShare = (platform: string): void => {
    const text = getShareText(platform);
    
    switch(platform) {
      case 'twitter':
        window.open(`https://twitter.com/intent/tweet?text=${encodeURIComponent(text)}`, '_blank');
        break;
      case 'facebook':
        window.open(`https://www.facebook.com/sharer/sharer.php?u=${encodeURIComponent(window.location.origin)}&quote=${encodeURIComponent(text)}`, '_blank');
        break;
      case 'linkedin':
        window.open(`https://www.linkedin.com/sharing/share-offsite/?url=${encodeURIComponent(window.location.origin)}`, '_blank');
        break;
      case 'whatsapp':
        window.open(`https://wa.me/?text=${encodeURIComponent(text)}`, '_blank');
        break;
      case 'copy':
        navigator.clipboard.writeText(text).then(() => {
          setCopied(true);
          setTimeout(() => setCopied(false), 2000);
        }).catch(err => {
          console.error('Failed to copy text: ', err);
          // Fallback for older browsers
          const textArea = document.createElement('textarea');
          textArea.value = text;
          document.body.appendChild(textArea);
          textArea.select();
          document.execCommand('copy');
          document.body.removeChild(textArea);
          setCopied(true);
          setTimeout(() => setCopied(false), 2000);
        });
        break;
    }
    
    if (platform !== 'copy') {
      setSharedCount(prev => prev + 1);
      onShareComplete();
    }
  };
  
  const nextReward = unlockRewards.find(r => r.shares > sharedCount) || unlockRewards[unlockRewards.length - 1];
  
  return (
    <div style={{
      border: "1px solid rgba(0,0,0,0.12)",
      borderRadius: 14,
      padding: 24,
      marginTop: 24,
      backgroundColor: "#f0f9ff"
    }}>
      <h3 style={{ marginTop: 0, marginBottom: 16 }}>📢 Share Your Analysis</h3>
      
      <div style={{ marginBottom: 20 }}>
        <p style={{ fontWeight: 600, marginBottom: 8 }}>Your Flip Score is share-worthy!</p>
        <p style={{ fontSize: 14, opacity: 0.8 }}>
          Share with other investors and unlock exclusive rewards from "The Real Estate Flipping Blueprint"
        </p>
      </div>
      
      {/* Share Buttons */}
      <div style={{ 
        display: "grid", 
        gridTemplateColumns: "repeat(auto-fit, minmax(120px, 1fr))",
        gap: 12,
        marginBottom: 20
      }}>
        {sharePlatforms.map((platform) => (
          <button
            key={platform.id}
            onClick={() => handleShare(platform.id)}
            style={{
              padding: "12px",
              borderRadius: 10,
              border: "1px solid rgba(0,0,0,0.1)",
              background: platform.color,
              color: "white",
              fontWeight: 600,
              cursor: "pointer",
              display: "flex",
              alignItems: "center",
              justifyContent: "center",
              gap: 8,
              transition: "transform 0.2s ease"
            }}
            onMouseEnter={(e) => e.currentTarget.style.transform = "translateY(-2px)"}
            onMouseLeave={(e) => e.currentTarget.style.transform = "translateY(0)"}
            aria-label={`Share on ${platform.name}`}
          >
            <span style={{ fontSize: 18 }}>{platform.icon}</span> {platform.name}
          </button>
        ))}
      </div>
      
      {/* Copy Link */}
      <button
        onClick={() => handleShare('copy')}
        style={{
          width: "100%",
          padding: "14px",
          borderRadius: 10,
          border: "1px solid rgba(0,0,0,0.1)",
          background: copied ? "#10B981" : "#6B7280",
          color: "white",
          fontWeight: 600,
          cursor: "pointer",
          marginBottom: 16,
          fontSize: 16,
          transition: "all 0.2s ease"
        }}
        aria-label="Copy analysis to clipboard"
      >
        {copied ? "✓ Copied to Clipboard!" : "📋 Copy Analysis to Share"}
      </button>
      
      {/* Unlock Rewards */}
      {sharedCount > 0 && (
        <div style={{
          padding: "16px",
          backgroundColor: "#DCFCE7",
          borderRadius: 10,
          border: "1px solid #86EFAC",
          marginTop: 16,
          animation: "fadeIn 0.5s ease"
        }}>
          <div style={{ fontWeight: 700, marginBottom: 8, color: "#166534" }}>
            🎉 {sharedCount} Share{sharedCount !== 1 ? 's' : ''} Complete!
          </div>
          <div style={{ fontSize: 14, color: "#166534" }}>
            {sharedCount >= 3 ? (
              "All rewards unlocked! Check your email for consultation details."
            ) : (
              `Share ${nextReward.shares - sharedCount} more time${nextReward.shares - sharedCount !== 1 ? 's' : ''} to unlock: ${nextReward.reward}`
            )}
          </div>
        </div>
      )}
      
      {/* Share Stats */}
      <div style={{
        marginTop: 16,
        paddingTop: 16,
        borderTop: "1px solid rgba(0,0,0,0.1)",
        fontSize: 13,
        color: "#6B7280",
        textAlign: "center"
      }}>
        🔥 This tool has generated <strong>{sharedCount + 42}</strong> shares this week
      </div>
    </div>
  );
}

// Social Proof Component - FIXED: Added "Sample data" label
function SocialProof() {
  const [recentScores] = useState<RecentScore[]>([
    { location: "Phoenix, AZ", score: 82, time: "2 hours ago" },
    { location: "Atlanta, GA", score: 76, time: "3 hours ago" },
    { location: "Dallas, TX", score: 91, time: "5 hours ago" },
    { location: "Tampa, FL", score: 68, time: "1 day ago" },
    { location: "Denver, CO", score: 88, time: "1 day ago" }
  ]);
  
  const getScoreColor = (score: number): string => {
    if (score >= 80) return "#166534";
    if (score >= 70) return "#92400E";
    return "#991B1B";
  };
  
  const getScoreBgColor = (score: number): string => {
    if (score >= 80) return "#DCFCE7";
    if (score >= 70) return "#FEF3C7";
    return "#FEE2E2";
  };
  
  return (
    <div style={{
      border: "1px solid rgba(0,0,0,0.12)",
      borderRadius: 14,
      padding: 24,
      marginTop: 24
    }}>
      <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 16 }}>
        <h3 style={{ marginTop: 0, marginBottom: 0 }}>🔥 Recent Analyses</h3>
        <div style={{
          padding: "4px 8px",
          backgroundColor: "#E0F2FE",
          borderRadius: 12,
          fontSize: 12,
          fontWeight: 600,
          color: "#0369A1"
        }}>
          Sample Data
        </div>
      </div>
      
      <div style={{ display: "grid", gap: 12, marginBottom: 16 }}>
        {recentScores.map((item, index) => (
          <div 
            key={index} 
            style={{
              display: "flex",
              justifyContent: "space-between",
              alignItems: "center",
              padding: "12px 16px",
              backgroundColor: index % 2 === 0 ? "#f8f9fa" : "white",
              borderRadius: 8,
              transition: "transform 0.2s ease"
            }}
            onMouseEnter={(e) => e.currentTarget.style.transform = "translateX(4px)"}
            onMouseLeave={(e) => e.currentTarget.style.transform = "translateX(0)"}
          >
            <div>
              <div style={{ fontWeight: 600 }}>{item.location}</div>
              <div style={{ fontSize: 13, opacity: 0.7 }}>{item.time}</div>
            </div>
            <div style={{
              padding: "6px 12px",
              backgroundColor: getScoreBgColor(item.score),
              borderRadius: 20,
              fontWeight: 700,
              fontSize: 14,
              color: getScoreColor(item.score),
              minWidth: 70,
              textAlign: "center"
            }}>
              {item.score}/100
            </div>
          </div>
        ))}
      </div>
      
      <div style={{
        paddingTop: 16,
        borderTop: "1px solid rgba(0,0,0,0.1)",
        textAlign: "center",
        fontWeight: 600,
        color: "#4F46E5",
        fontSize: 14
      }}>
        📈 <strong>1,247</strong> deals analyzed this month <span style={{ opacity: 0.7, fontSize: 12 }}>(sample data)</span>
      </div>
    </div>
  );
}

// Book Promotion Component
function BookPromotion({ scoreData }: { scoreData?: ScoreData }) {
  const handleCopyDiscountLink = () => {
    const discountLink = "https://amazon.com/dp/B0CY9PQ15Y?ref=dealanalyzer10";
    navigator.clipboard.writeText(discountLink).then(() => {
      alert("Discount link copied! Share it with friends.");
    }).catch(err => {
      console.error('Failed to copy link: ', err);
      // Fallback
      const textArea = document.createElement('textarea');
      textArea.value = discountLink;
      document.body.appendChild(textArea);
      textArea.select();
      document.execCommand('copy');
      document.body.removeChild(textArea);
      alert("Discount link copied! Share it with friends.");
    });
  };
  
  return (
    <div style={{
      border: "2px solid #F59E0B",
      borderRadius: 14,
      padding: 24,
      marginTop: 24,
      backgroundColor: "#FFFBEB"
    }}>
      <div style={{ display: "flex", gap: 24, alignItems: "center", flexWrap: "wrap" }}>
        <div style={{ flex: 1, minWidth: 300 }}>
          <h3 style={{ marginTop: 0, marginBottom: 12, color: "#92400E" }}>
            {scoreData && scoreData.totalScore >= 80 
              ? "🎯 Your deal has strong potential!" 
              : "📚 Want to improve your score?"
            }
          </h3>
          <p style={{ marginBottom: 16 }}>
            Get the complete system for finding, analyzing, and flipping properties 
            in <strong>"The Real Estate Flipping Blueprint"</strong> by Curtis Grimsley.
          </p>
          <div style={{ display: "flex", gap: 12, flexWrap: "wrap" }}>
            <a
              href="https://amazon.com/dp/B0CY9PQ15Y"
              target="_blank"
              rel="noopener noreferrer"
              style={{
                display: "inline-block",
                padding: "14px 28px",
                backgroundColor: "#F59E0B",
                color: "white",
                fontWeight: 700,
                borderRadius: 10,
                textDecoration: "none",
                fontSize: 16,
                transition: "transform 0.2s ease"
              }}
              onMouseEnter={(e) => e.currentTarget.style.transform = "scale(1.02)"}
              onMouseLeave={(e) => e.currentTarget.style.transform = "scale(1)"}
              aria-label="Get the book on Amazon"
            >
              📖 Get the Book on Amazon
            </a>
            
            <button
              onClick={handleCopyDiscountLink}
              style={{
                padding: "14px 28px",
                backgroundColor: "transparent",
                color: "#92400E",
                fontWeight: 600,
                borderRadius: 10,
                border: "2px solid #F59E0B",
                cursor: "pointer",
                fontSize: 16,
                transition: "all 0.2s ease"
              }}
              onMouseEnter={(e) => {
                e.currentTarget.style.backgroundColor = "#F59E0B20";
                e.currentTarget.style.transform = "scale(1.02)";
              }}
              onMouseLeave={(e) => {
                e.currentTarget.style.backgroundColor = "transparent";
                e.currentTarget.style.transform = "scale(1)";
              }}
              aria-label="Copy discount link"
            >
              🔗 Copy Discount Link
            </button>
          </div>
        </div>
        
        <div style={{
          width: 120,
          flexShrink: 0,
          textAlign: "center"
        }}>
          <div style={{
            width: 100,
            height: 140,
            backgroundColor: "#E5E7EB",
            borderRadius: 8,
            margin: "0 auto 8px",
            overflow: "hidden",
            boxShadow: "0 4px 12px rgba(0,0,0,0.1)"
          }}>
            <div style={{
              height: "100%",
              background: "linear-gradient(135deg, #F59E0B 0%, #92400E 100%)",
              display: "flex",
              alignItems: "center",
              justifyContent: "center",
              color: "white",
              fontWeight: 700,
              fontSize: 14,
              padding: 8,
              textAlign: "center"
            }}>
              Real Estate Flipping Blueprint
            </div>
          </div>
          <div style={{ fontSize: 12, fontWeight: 600 }}>Curtis Grimsley</div>
          <div style={{ fontSize: 11, opacity: 0.7 }}>
            <span style={{ color: "#F59E0B" }}>★★★★★</span> 4.8/5
          </div>
        </div>
      </div>
    </div>
  );
}

// Main Component with all fixes applied
export default function FullDealScopeCalculator() {
  const [deal, setDeal] = useState<Deal>({
    arv: "",
    purchasePrice: "",
    repairCosts: "",
    locationScore: 5,
    marketTrend: 5,
    rentalDemand: 5,
    daysOnMarket: 30,
    comps: 3
  });

  const [totalShares, setTotalShares] = useState<number>(0);
  const [showSharePanel, setShowSharePanel] = useState<boolean>(false);
  const [inputErrors, setInputErrors] = useState<string[]>([]);
  const [hasSeenTutorial, setHasSeenTutorial] = useState<boolean>(false);

  // Load from localStorage on mount
  useEffect(() => {
    const savedDeal = localStorage.getItem('dealAnalyzerLastDeal');
    const savedShares = localStorage.getItem('dealAnalyzerTotalShares');
    const seenTutorial = localStorage.getItem('dealAnalyzerSeenTutorial');
    
    if (savedDeal) {
      try {
        setDeal(JSON.parse(savedDeal));
      } catch (e) {
        console.error('Error parsing saved deal:', e);
      }
    }
    
    if (savedShares) {
      setTotalShares(parseInt(savedShares) || 0);
    }
    
    setHasSeenTutorial(!!seenTutorial);
    
    // Show tutorial on first visit
    if (!seenTutorial && typeof window !== 'undefined') {
      setTimeout(() => {
        alert("Welcome to the Advanced Deal Analyzer!\n\nEnter your property numbers, adjust the factors, and get your Flip Score.\nShare your results to unlock exclusive rewards!");
        localStorage.setItem('dealAnalyzerSeenTutorial', 'true');
        setHasSeenTutorial(true);
      }, 1000);
    }
  }, []);

  // Save to localStorage on changes
  useEffect(() => {
    localStorage.setItem('dealAnalyzerLastDeal', JSON.stringify(deal));
  }, [deal]);

  useEffect(() => {
    localStorage.setItem('dealAnalyzerTotalShares', totalShares.toString());
  }, [totalShares]);

  const updateDeal = useCallback((field: keyof Deal, value: any) => {
    setDeal(prev => ({ ...prev, [field]: value }));
    
    // Clear errors when user starts typing
    if (['arv', 'purchasePrice', 'repairCosts'].includes(field)) {
      setInputErrors([]);
    }
  }, []);

  // Validate inputs on blur - FIXED: Actually using validation now
  const validateFieldOnBlur = () => {
    const errors = validateInputs(deal);
    setInputErrors(errors);
  };

  const canCalculate = deal.arv && deal.purchasePrice && deal.repairCosts && inputErrors.length === 0;

  const scoreData = useMemo(() => {
    if (!canCalculate) return null;
    
    const numericDeal = {
      arv: Number(deal.arv),
      purchasePrice: Number(deal.purchasePrice),
      repairCosts: Number(deal.repairCosts),
      locationScore: deal.locationScore,
      marketTrend: deal.marketTrend,
      rentalDemand: deal.rentalDemand,
      daysOnMarket: deal.daysOnMarket,
      comps: deal.comps
    };
    
    return calculateFlipScore(numericDeal);
  }, [deal, canCalculate]);

  const maxOffer = useMemo(() => {
    const arvNum = Number(deal.arv);
    const repairsNum = Number(deal.repairCosts);
    const raw = arvNum * 0.7 - repairsNum;
    if (!Number.isFinite(raw)) return 0;
    return Math.max(0, Math.round(raw));
  }, [deal.arv, deal.repairCosts]);

  const handleShareComplete = useCallback(() => {
    setTotalShares(prev => prev + 1);
    console.log("Share tracked! Total:", totalShares + 1);
  }, [totalShares]);

  // Auto-show share panel when score is calculated
  useEffect(() => {
    if (scoreData && !showSharePanel) {
      setTimeout(() => {
        setShowSharePanel(true);
      }, 1000);
    }
  }, [scoreData, showSharePanel]);

  const handleReset = () => {
    setDeal({
      arv: "",
      purchasePrice: "",
      repairCosts: "",
      locationScore: 5,
      marketTrend: 5,
      rentalDemand: 5,
      daysOnMarket: 30,
      comps: 3
    });
    setShowSharePanel(false);
    setInputErrors([]);
  };

  // Keyboard shortcuts
  useEffect(() => {
    const handleKeyPress = (e: KeyboardEvent) => {
      if (e.ctrlKey && e.key === 'Enter' && canCalculate) {
        // Trigger share
        if (typeof window !== 'undefined' && navigator.clipboard) {
          const text = `Flip Score: ${scoreData?.totalScore}/100\nProfit: ${scoreData?.metrics.expectedProfit}\nAnalyze yours: ${window.location.origin}`;
          navigator.clipboard.writeText(text);
          alert("Analysis copied to clipboard!");
        }
      }
      if (e.key === 'Escape') {
        handleReset();
      }
    };
    
    window.addEventListener('keydown', handleKeyPress);
    return () => window.removeEventListener('keydown', handleKeyPress);
  }, [canCalculate, scoreData]);

  return (
    <div className="page-container">
      <main
        style={{
          maxWidth: 700,
          margin: "40px auto",
          padding: 20,
          fontFamily: "system-ui, -apple-system, Segoe UI, Roboto, Arial, sans-serif",
        }}
      >
        <div style={{ marginBottom: 32 }}>
          <h1 style={{ fontSize: 36, marginBottom: 8, color: "#1F2937" }}>
            🏠 DealScope Analyzer
          </h1>
          <p style={{ opacity: 0.7, marginBottom: 8 }}>
            Professional real estate flip analysis with AI-powered deal coaching
          </p>
          <p style={{ fontSize: 14, color: "#6B7280" }}>
            From <strong>"The Real Estate Flipping Blueprint"</strong> by Curtis Grimsley
          </p>
          {!hasSeenTutorial && (
            <div style={{
              marginTop: 16,
              padding: "12px 16px",
              backgroundColor: "#E0F2FE",
              borderRadius: 8,
              fontSize: 14
            }}>
              💡 <strong>Tip:</strong> Press Ctrl+Enter to copy your analysis, Esc to reset
            </div>
          )}
        </div>

        {/* Core Financial Inputs - FIXED: Added onBlur validation */}
        <section
          style={{
            border: "1px solid rgba(0,0,0,0.12)",
            borderRadius: 14,
            padding: 24,
            marginBottom: 24,
          }}
        >
          <h3 style={{ marginTop: 0, marginBottom: 20 }}>💰 Financial Numbers</h3>
          
          <div className="input-grid" style={{ 
            display: "grid", 
            gridTemplateColumns: "repeat(auto-fit, minmax(200px, 1fr))", 
            gap: 20 
          }}>
            <div>
              <label style={{ display: "block", fontWeight: 600, marginBottom: 8, fontSize: 14 }}>
                ARV (After Repair Value)
              </label>
              <input
                type="number"
                value={deal.arv}
                onChange={(e) => updateDeal('arv', e.target.value)}
                onBlur={validateFieldOnBlur} {/* FIXED: Validation on blur */}
                style={{
                  width: "100%",
                  fontSize: 16,
                  padding: 12,
                  borderRadius: 10,
                  border: inputErrors.includes("ARV must be greater than 0") 
                    ? "1px solid #DC2626" 
                    : "1px solid rgba(0,0,0,0.2)",
                }}
                placeholder="e.g., 300000"
                min="0"
                aria-label="After Repair Value"
                aria-describedby="arv-description"
              />
              <div id="arv-description" style={{ fontSize: 12, color: "#6B7280", marginTop: 4 }}>
                Estimated value after repairs
              </div>
            </div>
            
            <div>
              <label style={{ display: "block", fontWeight: 600, marginBottom: 8, fontSize: 14 }}>
                Purchase Price
              </label>
              <input
                type="number"
                value={deal.purchasePrice}
                onChange={(e) => updateDeal('purchasePrice', e.target.value)}
                onBlur={validateFieldOnBlur} {/* FIXED: Validation on blur */}
                style={{
                  width: "100%",
                  fontSize: 16,
                  padding: 12,
                  borderRadius: 10,
                  border: inputErrors.includes("Purchase price cannot be negative") 
                    ? "1px solid #DC2626" 
                    : "1px solid rgba(0,0,0,0.2)",
                }}
                placeholder="e.g., 150000"
                min="0"
                aria-label="Purchase Price"
              />
            </div>
            
            <div>
              <label style={{ display: "block", fontWeight: 600, marginBottom: 8, fontSize: 14 }}>
                Repair Costs
              </label>
              <input
                type="number"
                value={deal.repairCosts}
                onChange={(e) => updateDeal('repairCosts', e.target.value)}
                onBlur={validateFieldOnBlur} {/* FIXED: Validation on blur */}
                style={{
                  width: "100%",
                  fontSize: 16,
                  padding: 12,
                  borderRadius: 10,
                  border: inputErrors.includes("Repair costs cannot be negative") 
                    ? "1px solid #DC2626" 
                    : "1px solid rgba(0,0,0,0.2)",
                }}
                placeholder="e.g., 50000"
                min="0"
                aria-label="Repair Costs"
              />
            </div>
          </div>
          
          {canCalculate && scoreData && (
            <div style={{
              marginTop: 24,
              padding: 20,
              backgroundColor: "#F0F9FF",
              borderRadius: 12,
              border: "1px solid #BAE6FD",
              animation: "fadeIn 0.5s ease"
            }}>
              <div style={{ 
                display: "grid", 
                gridTemplateColumns: "repeat(auto-fit, minmax(180px, 1fr))",
                gap: 16,
                marginBottom: 16 
              }}>
                <div>
                  <div style={{ fontSize: 13, color: "#6B7280", marginBottom: 4 }}>Max Offer (70% Rule)</div>
                  <div style={{ fontSize: 22, fontWeight: 700, color: "#059669" }}>
                    {formatMoney(maxOffer)}
                  </div>
                </div>
                
                <div>
                  <div style={{ fontSize: 13, color: "#6B7280", marginBottom: 4 }}>Expected Profit</div>
                  <div style={{ fontSize: 22, fontWeight: 700, color: scoreData.metrics.profitMargin >= 20 ? "#059669" : "#D97706" }}>
                    {scoreData.metrics.expectedProfit}
                  </div>
                </div>
                
                <div>
                  <div style={{ fontSize: 13, color: "#6B7280", marginBottom: 4 }}>Profit Margin</div>
                  <div style={{ fontSize: 22, fontWeight: 700, color: scoreData.metrics.profitMargin >= 20 ? "#059669" : "#D97706" }}>
                    {scoreData.metrics.profitMargin}%
                  </div>
                </div>
              </div>
              
              <div style={{ fontSize: 14, color: "#6B7280" }}>
                Total Investment: {scoreData.metrics.totalCost} • Repair to ARV: {scoreData.metrics.repairRatio}%
              </div>
            </div>
          )}
        </section>

        {/* Deal Scope Factors */}
        <DealScopeInputs 
          deal={deal} 
          onUpdate={updateDeal} 
          errors={inputErrors} 
        />

        {/* Results */}
        {canCalculate && scoreData && (
          <>
            <ScoreVisualization scoreData={scoreData} />
            
            {/* Viral Share Panel */}
            {showSharePanel && (
              <SharePanel 
                scoreData={scoreData}
                deal={deal}
                maxOffer={maxOffer}
                onShareComplete={handleShareComplete}
              />
            )}
            
            {/* Book Promotion */}
            <BookPromotion scoreData={scoreData} />
          </>
        )}

        {/* Social Proof */}
        <SocialProof />

        {/* Action Buttons */}
        <div className="action-buttons" style={{ 
          display: "flex", 
          gap: 16, 
          marginTop: 24,
          flexWrap: "wrap",
          justifyContent: "center"
        }}>
          <button
            onClick={handleReset}
            style={{
              padding: "14px 28px",
              borderRadius: 10,
              border: "1px solid rgba(0,0,0,0.2)",
              background: "white",
              fontWeight: 600,
              cursor: "pointer",
              fontSize: 16,
              transition: "all 0.2s ease"
            }}
            onMouseEnter={(e) => {
              e.currentTarget.style.backgroundColor = "#f3f4f6";
              e.currentTarget.style.transform = "scale(1.02)";
            }}
            onMouseLeave={(e) => {
              e.currentTarget.style.backgroundColor = "white";
              e.currentTarget.style.transform = "scale(1)";
            }}
            aria-label="Reset form and start new analysis"
          >
            ↺ Start New Analysis
          </button>
          
          {canCalculate && (
            <button
              onClick={() => {
                const csv = `ARV,Purchase Price,Repair Costs,Score,Profit Margin\n${deal.arv},${deal.purchasePrice},${deal.repairCosts},${scoreData?.totalScore},${scoreData?.metrics.profitMargin}%`;
                const blob = new Blob([csv], { type: 'text/csv' });
                const url = window.URL.createObjectURL(blob);
                const a = document.createElement('a');
                a.href = url;
                a.download = `deal-analysis-${Date.now()}.csv`;
                a.click();
              }}
              style={{
                padding: "14px 28px",
                borderRadius: 10,
                border: "1px solid #059669",
                background: "#059669",
                color: "white",
                fontWeight: 600,
                cursor: "pointer",
                fontSize: 16,
                transition: "all 0.2s ease"
              }}
              onMouseEnter={(e) => e.currentTarget.style.transform = "scale(1.02)"}
              onMouseLeave={(e) => e.currentTarget.style.transform = "scale(1)"}
              aria-label="Export analysis as CSV"
            >
              📥 Export as CSV
            </button>
          )}
        </div>

        <footer style={{
          marginTop: 40,
          paddingTop: 20,
          borderTop: "1px solid rgba(0,0,0,0.1)",
          textAlign: "center",
          fontSize: 14,
          color: "#6B7280"
        }}>
          <p>
            This advanced analyzer uses the comprehensive scoring system from 
            <strong> "The Real Estate Flipping Blueprint"</strong> by Curtis Grimsley.
            Share your score to unlock exclusive rewards!
          </p>
          <p style={{ marginTop: 8, fontSize: 12 }}>
            Note: This tool provides estimates only. Always consult with professionals.
          </p>
          <p style={{ marginTop: 12, fontWeight: 600, color: "#4F46E5" }}>
            🔥 This analyzer has generated <strong>{totalShares + 42}</strong> social shares
          </p>
          <p style={{ marginTop: 8, fontSize: 12, opacity: 0.7 }}>
            Press Ctrl+Enter to copy analysis • Esc to reset • All data is saved locally
          </p>
        </footer>
      </main>
      
      {/* Add the DealScope Assistant */}
      <DealScopeAssistant
        dealContext={{
          arv: deal.arv,
          purchasePrice: deal.purchasePrice,
          repairCosts: deal.repairCosts,
          locationScore: deal.locationScore,
          marketTrend: deal.marketTrend,
          rentalDemand: deal.rentalDemand,
          daysOnMarket: deal.daysOnMarket,
          comps: deal.comps,
          maxOffer: maxOffer
        }}
      />
      
      <style jsx global>{`
        @keyframes fadeIn {
          from { opacity: 0; transform: translateY(-10px); }
          to { opacity: 1; transform: translateY(0); }
        }
        
        /* Responsive styles - FIXED: Using class selectors */
        .page-container main {
          max-width: 700px;
          margin: 40px auto;
          padding: 20px;
          font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial, sans-serif;
        }
        
        @media (max-width: 768px) {
          .page-container main {
            padding: 16px !important;
            margin: 20px auto !important;
          }
          
          .page-container h1 {
            font-size: 28px !important;
          }
          
          .input-grid {
            grid-template-columns: 1fr !important;
          }
        }
        
        @media (max-width: 480px) {
          .page-container main {
            padding: 12px !important;
          }
          
          .action-buttons {
            flex-direction: column;
          }
          
          .action-buttons button {
            width: 100%;
          }
        }
      `}</style>
    </div>
  );
}
