/**
 * n8n Code node
 * - AI 결과에서 팀명 추출 → 최종 문장 생성 → 지정된 웹훅으로 POST 전송
 * - 다음 노드(Google Sheets 또는 Gmail)에서 team 값을 사용할 수 있도록 반환
 */

const WEBHOOK_URL = 'http://localhost:5678/webhook/61446b74-ef74-4de4-a733-71e35fa65794/chat';

// ----------------------------
// 1) 팀명 추출 로직
// ----------------------------

// AI Agent가 반환한 문장 (예: "문의해주신 민원은 시설관리팀으로 전달되었습니다. 감사합니다.")
const rawMessage = $json.message || $json.text || "";

let team = $json.selectedTeam ?? $json.team ?? null;

// 정규식으로 문장 속 팀명 자동 추출
if (!team) {
  const match = rawMessage.match(/문의해주신\s*민원은\s*(.*?)으로\s*전달되었습니다/);
  team = match ? match[1].trim() : null;
}

// 기본값 설정
if (!team) team = '총무팀';
if (!team.endsWith('팀')) team += '팀';

// ----------------------------
// 2) 최종 메시지 생성
// ----------------------------
const message = `문의해주신 민원은 ${team}으로 전달되었습니다. 감사합니다.`;

// ----------------------------
// 3) 웹훅으로 전송
// ----------------------------
let responseBody;
try {
  responseBody = await this.helpers.httpRequest({
    method: 'POST',
    url: WEBHOOK_URL,
    headers: { 'Content-Type': 'application/json' },
    body: { message },
    json: true,
  });
  console.log('[OK] Sent to webhook:', message);
} catch (err) {
  console.log('[ERROR] Failed to send to webhook:', err?.message || err);
  // 실패하더라도 다음 노드에서 확인 가능하도록 반환
  return [
    {
      json: {
        message,
        team,
        error: String(err?.message || err),
      },
    },
  ];
}

// ----------------------------
// 4) 다음 노드(Google Sheets, Gmail 등)에 전달할 데이터 반환
// ----------------------------
return [
  {
    json: {
      team, // Gmail node에서 부서명으로 이메일 조회 가능
      message,
      webhookResponse: responseBody,
    },
  },
];
