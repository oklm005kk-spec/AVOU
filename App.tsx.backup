import "react-native-url-polyfill/auto";
import React, { useEffect, useMemo, useRef, useState } from "react";
import {
  Alert,
  AppState,
  Animated,
  Easing,
  Share,
  KeyboardAvoidingView,
  Modal,
  Platform,
  SafeAreaView,
  ScrollView,
  StatusBar,
  StyleSheet,
  Text,
  TextInput,
  TouchableOpacity,
  View,
} from "react-native";
import { createClient, RealtimeChannel } from "@supabase/supabase-js";
import * as Clipboard from "expo-clipboard";

/**
 * AVOU — ACTION OU VÉRITÉ
 *
 * RÈGLES IMPORTANTES
 * - Aucun écran "Se connecter".
 * - Aucun système de création de compte. Le jeu est sans compte.
 * - Le créateur est le Maître de jeu.
 * - Le Maître de jeu contrôle les identités : révélées ou cachées.
 * - MODE INTENSE : option « Tentative de démasquage ». Après chaque tour,
 *   tous les joueurs votent pour deviner l'auteur de la proposition choisie.
 *   La cible vote aussi, puis le résultat est révélé avant le tour suivant.
 * - Le Maître de jeu contrôle Ordre / Désordre.
 * - Ordre = séquence fixe.
 * - Désordre = l'application choisit aléatoirement le prochain joueur.
 * - MODE INTENSE : à chaque tour, les joueurs autres que la cible écrivent 1 Action
 *   ET 1 Vérité. Les anciennes propositions sont supprimées.
 * - MODE CIBLE : le joueur actif choisit directement un autre joueur et lui envoie
 *   une seule Action OU une seule Vérité. La cible ne voit JAMAIS le contenu avant
 *   son choix : elle voit uniquement le DOS indiquant ACTION ou VÉRITÉ.
 * - Après le choix, la carte se retourne et son contenu est révélé.
 * - Aucune sauvegarde d'Action/Vérité pour un tour suivant.
 * - MESSAGERIE PRIVÉE : chaque joueur peut écrire à n'importe quel autre joueur.
 *   Chaque conversation utilise son propre canal Realtime entre les deux joueurs.
 *   Les messages privés ne sont jamais placés dans GAME_STATE et ne sont donc
 *   pas envoyés au canal public de la partie.
 *
 * Supabase :
 * remplace SUPABASE_ANON_KEY par ta clé publique anon/publishable.
 * Ne mets JAMAIS une clé service_role dans l'application.
 */

const SUPABASE_URL = "https://iurrbuqvjaifmkaggpbw.supabase.co";
const SUPABASE_ANON_KEY =
  process.env.EXPO_PUBLIC_SUPABASE_ANON_KEY ??
  "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6Iml1cnJidXF2amFpZm1rYWdncGJ3Iiwicm9sZSI6ImFub24iLCJpYXQiOjE3OTA5MDg0MTgsImV4cCI6MjEwNjQ4NDQxOH0.KEybIgMVHdOCtoTijZHXnG1kF93Uc5itWUH4j49mapU";

const supabase =
  SUPABASE_ANON_KEY.startsWith("COLLE_")
    ? null
    : createClient(SUPABASE_URL, SUPABASE_ANON_KEY, {
        auth: { persistSession: false, autoRefreshToken: false },
      });

type Lang = "fr" | "en" | "es" | "it" | "de";
type Screen =
  | "welcome"
  | "home"
  | "create"
  | "join"
  | "language"
  | "avatar"
  | "space"
  | "game";

type Phase =
  | "lobby"
  | "settings"
  | "orderChoice"
  | "preparation"
  | "play"
  | "reveal"
  | "demaskingVote"
  | "demaskingResult"
  | "roundEnd"
  | "gameEnd";

type IdentityMode = "reveal" | "hidden";
type OrderMode = "order" | "disorder";
type GameMode = "target" | "intense";

type Player = {
  id: string;
  name: string;
  avatar: string;
};

type Proposal = {
  id: string;
  type: "action" | "truth";
  text: string;
  authorId: string;
};

type GameState = {
  code: string;
  hostId: string;
  players: Player[];
  joinRequests: Player[];
  phase: Phase;

  round: number;
  maxRounds: number;

  identityMode: IdentityMode;
  roundIdentity: "reveal" | "hidden";

  orderMode: OrderMode | null;
  gameMode: GameMode;
  targetPlayerId: string | null;
  turnIndex: number;
  currentPlayerId: string;

  // Propositions ONLY for the current turn.
  proposals: Proposal[];
  submittedForTurn: string[];

  selectedProposalId: string | null;
  revealed: boolean;

  // Tentative de démasquage — uniquement après un tour Intense.
  demaskingEnabled: boolean;
  demaskingVotes: Record<string, string>;
  demaskingResult: string | null;
  demaskingTie: boolean;

  score: Record<string, number>;
  stateVersion: number;

};

type Action =
  | { type: "JOIN_REQUEST"; player: Player }
  | { type: "APPROVE_JOIN"; playerId: string; requesterId: string }
  | { type: "REJECT_JOIN"; playerId: string; requesterId: string }
  | { type: "JOIN"; player: Player }
  | { type: "START_SETUP"; playerId: string }
  | { type: "SET_IDENTITY_MODE"; playerId: string; mode: IdentityMode }
  | { type: "SET_DEMASKING_ENABLED"; playerId: string; enabled: boolean }
  | { type: "SET_ORDER"; playerId: string; mode: OrderMode }
  | { type: "SET_GAME_MODE"; playerId: string; mode: GameMode }
  | { type: "UPDATE_GAME_SETTINGS"; playerId: string; identityMode: IdentityMode; gameMode: GameMode; orderMode: OrderMode }
  | { type: "START_ORDER_CHOICE"; playerId: string }
  | {
      type: "SUBMIT_PROPOSALS";
      playerId: string;
      actionText: string;
      truthText: string;
    }
  | {
      type: "SUBMIT_TARGET_PROPOSAL";
      playerId: string;
      targetPlayerId: string;
      typeChoice: "action" | "truth";
      text: string;
    }
  | { type: "SELECT_PROPOSAL"; playerId: string; proposalId: string }
  | { type: "REVEAL_PROPOSAL"; playerId: string }
  | { type: "NEXT_TURN"; playerId: string }
  | { type: "SUBMIT_DEMASKING_VOTE"; playerId: string; authorId: string }
  | { type: "CONTINUE_AFTER_DEMASKING"; playerId: string }
  | { type: "NEXT_ROUND"; playerId: string }
  | { type: "LEAVE"; playerId: string }
  | { type: "DISCONNECT_PLAYER"; playerId: string; targetPlayerId: string }
  | { type: "REMOVE_PLAYER"; playerId: string; targetPlayerId: string }
  | { type: "ADD_PLAYER_DIRECT"; playerId: string; name: string };

const uid = () => Math.random().toString(36).slice(2, 10);

function generateCode() {
  const chars = "ABCDEFGHJKLMNPQRSTUVWXYZ23456789";
  return Array.from(
    { length: 4 },
    () => chars[Math.floor(Math.random() * chars.length)]
  ).join("");
}

function randomNextPlayer(state: GameState): { id: string; index: number } {
  const candidates = state.players
    .map((p, index) => ({ id: p.id, index }))
    .filter((x) => x.id !== state.currentPlayerId);

  const list = candidates.length ? candidates : state.players.map((p, index) => ({
    id: p.id,
    index,
  }));

  const pick = list[Math.floor(Math.random() * list.length)];
  return pick ?? { id: state.currentPlayerId, index: state.turnIndex };
}

// Sélection pseudo-aléatoire mais déterministe : tous les appareils calculent
// le même nouveau Maître de jeu à partir de la salle et du joueur sortant.
// Cela évite que deux téléphones élisent deux hôtes différents.
function deterministicHostId(state: GameState, leavingPlayerId: string, candidates = state.players.filter((p) => p.id !== leavingPlayerId)): string {
  if (candidates.length === 0) return "";
  const seed = `${state.code}:${leavingPlayerId}:${candidates.map((p) => p.id).sort().join(",")}`;
  let hash = 2166136261;
  for (let i = 0; i < seed.length; i += 1) {
    hash ^= seed.charCodeAt(i);
    hash = Math.imul(hash, 16777619);
  }
  const index = Math.abs(hash) % candidates.length;
  return candidates[index]?.id ?? candidates[0].id;
}

function makeInitialState(code: string, host: Player): GameState {
  return {
    code,
    hostId: host.id,
    players: [host],
    joinRequests: [],
    phase: "lobby",

    round: 1,
    maxRounds: 5,

    identityMode: "reveal",
    roundIdentity: "reveal",

    orderMode: null,
    gameMode: "target",
    targetPlayerId: null,
    turnIndex: 0,
    currentPlayerId: host.id,

    proposals: [],
    submittedForTurn: [],

    selectedProposalId: null,
    revealed: false,

    demaskingEnabled: false,
    demaskingVotes: {},
    demaskingResult: null,
    demaskingTie: false,

    score: { [host.id]: 0 },
    stateVersion: 1,
  };
}

function minimumPlayersForMode(mode: GameMode): number {
  return mode === "target" ? 2 : 3;
}

function removePlayerFromState(state: GameState, playerId: string, forcedHostId?: string): GameState {
  if (!state.players.some((p) => p.id === playerId)) return state;

  const remaining = state.players.filter((p) => p.id !== playerId);
  const removedWasCurrent = state.currentPlayerId === playerId;
  const nextHostId = state.hostId === playerId
    ? (forcedHostId && remaining.some((p) => p.id === forcedHostId)
        ? forcedHostId
        : deterministicHostId(state, playerId, remaining))
    : state.hostId;

  if (remaining.length <= 1) {
    return {
      ...state,
      players: remaining,
      hostId: nextHostId,
      phase: "gameEnd",
      currentPlayerId: remaining[0]?.id ?? "",
      turnIndex: 0,
      joinRequests: state.joinRequests.filter((p) => p.id !== playerId),
      proposals: [],
      submittedForTurn: [],
      selectedProposalId: null,
      revealed: false,
      score: Object.fromEntries(Object.entries(state.score).filter(([id]) => id !== playerId)),
    };
  }

  const nextMode: GameMode = state.gameMode === "intense" && remaining.length < 3 ? "target" : state.gameMode;
  let nextCurrentId = state.currentPlayerId;
  let nextTurnIndex = remaining.findIndex((p) => p.id === nextCurrentId);

  if (removedWasCurrent || nextTurnIndex < 0) {
    const next = remaining[0];
    nextCurrentId = next?.id ?? "";
    nextTurnIndex = Math.max(0, remaining.findIndex((p) => p.id === nextCurrentId));
  }

  const submitted = state.submittedForTurn.filter((id) => id !== playerId);
  const required = remaining.filter((p) => p.id !== nextCurrentId).length;
  const preparationComplete = nextMode === "intense" && state.phase === "preparation" && submitted.length >= required;

  return {
    ...state,
    players: remaining,
    hostId: nextHostId,
    gameMode: nextMode,
    currentPlayerId: nextCurrentId,
    turnIndex: nextTurnIndex,
    targetPlayerId: state.targetPlayerId === playerId ? null : state.targetPlayerId,
    proposals: removedWasCurrent || nextMode !== state.gameMode ? [] : state.proposals.filter((p) => p.authorId !== playerId),
    submittedForTurn: nextMode !== state.gameMode ? [] : submitted,
    selectedProposalId: removedWasCurrent || nextMode !== state.gameMode ? null : state.selectedProposalId,
    revealed: removedWasCurrent || nextMode !== state.gameMode ? false : state.revealed,
    phase: nextMode !== state.gameMode ? "preparation" : (preparationComplete ? "play" : (removedWasCurrent && !["lobby", "settings", "orderChoice"].includes(state.phase) ? "preparation" : state.phase)),
    joinRequests: state.joinRequests.filter((p) => p.id !== playerId),
    score: Object.fromEntries(Object.entries(state.score).filter(([id]) => id !== playerId)),
  };
}

function reducer(state: GameState, action: Action): GameState {
  switch (action.type) {
    case "JOIN_REQUEST": {
      if (state.players.some((p) => p.id === action.player.id)) return state;
      if (state.joinRequests.some((p) => p.id === action.player.id)) return state;
      return { ...state, joinRequests: [...state.joinRequests, action.player] };
    }

    case "APPROVE_JOIN": {
      if (action.playerId !== state.hostId) return state;
      const requester = state.joinRequests.find((p) => p.id === action.requesterId);
      if (!requester) return state;
      return {
        ...state,
        players: state.players.some((p) => p.id === requester.id) ? state.players : [...state.players, requester],
        joinRequests: state.joinRequests.filter((p) => p.id !== requester.id),
        score: { ...state.score, [requester.id]: state.score[requester.id] ?? 0 },
      };
    }

    case "REJECT_JOIN": {
      if (action.playerId !== state.hostId) return state;
      return { ...state, joinRequests: state.joinRequests.filter((p) => p.id !== action.requesterId) };
    }

    case "JOIN": {
      return state;
    }

    case "START_SETUP": {
      if (action.playerId !== state.hostId) return state;
      if (state.players.length < minimumPlayersForMode(state.gameMode)) return state;
      return { ...state, phase: "settings" };
    }

    case "SET_IDENTITY_MODE": {
      if (state.phase !== "settings") return state;
      if (action.playerId !== state.hostId) return state;

      return {
        ...state,
        identityMode: action.mode,
        roundIdentity: action.mode === "hidden" ? "hidden" : "reveal",
      };
    }

    case "SET_DEMASKING_ENABLED": {
      if (action.playerId !== state.hostId) return state;
      if (state.gameMode !== "intense") return { ...state, demaskingEnabled: false };
      if (!["lobby", "settings", "orderChoice", "preparation", "play", "reveal", "demaskingVote", "demaskingResult"].includes(state.phase)) return state;
      return {
        ...state,
        demaskingEnabled: action.enabled,
        demaskingVotes: {},
        demaskingResult: null,
        demaskingTie: false,
      };
    }

    case "SET_GAME_MODE": {
      if (action.playerId !== state.hostId) return state;
      if (state.phase === "gameEnd") return state;
      if (state.players.length < minimumPlayersForMode(action.mode)) return state;

      const modeChanged = state.gameMode !== action.mode;
      const nextIdentityMode: IdentityMode = action.mode === "target" ? "reveal" : state.identityMode;
      return {
        ...state,
        gameMode: action.mode,
        identityMode: nextIdentityMode,
        roundIdentity: action.mode === "target" ? "reveal" : state.roundIdentity,
        demaskingEnabled: action.mode === "target" ? false : state.demaskingEnabled,
        ...(modeChanged && !["lobby", "settings", "orderChoice"].includes(state.phase)
          ? {
              phase: "preparation" as const,
              proposals: [],
              submittedForTurn: [],
              targetPlayerId: null,
              selectedProposalId: null,
              revealed: false,
            }
          : {}),
      };
    }

    case "UPDATE_GAME_SETTINGS": {
      // Pendant la partie, seul le Maître de jeu peut modifier les réglages.
      if (action.playerId !== state.hostId) return state;
      if (state.phase === "gameEnd") return state;
      if (state.players.length < minimumPlayersForMode(action.gameMode)) return state;

      const modeChanged = state.gameMode !== action.gameMode;
      const nextIdentityMode: IdentityMode = action.gameMode === "target" ? "reveal" : action.identityMode;

      return {
        ...state,
        identityMode: nextIdentityMode,
        roundIdentity:
          action.gameMode === "target"
            ? "reveal"
            : action.identityMode === "hidden"
              ? "hidden"
              : action.identityMode === "reveal"
                ? "reveal"
                : state.roundIdentity,
        gameMode: action.gameMode,
        demaskingEnabled: action.gameMode === "target" ? false : state.demaskingEnabled,
        demaskingVotes: {},
        demaskingResult: null,
        demaskingTie: false,
        orderMode: action.orderMode,
        ...(modeChanged && !["lobby", "settings", "orderChoice"].includes(state.phase)
          ? {
              phase: "preparation" as const,
              proposals: [],
              submittedForTurn: [],
              targetPlayerId: null,
              selectedProposalId: null,
              revealed: false,
            }
          : {}),
      };
    }

    case "START_ORDER_CHOICE": {
      if (state.phase !== "settings") return state;
      if (action.playerId !== state.hostId) return state;
      return { ...state, phase: "orderChoice" };
    }

    case "SET_ORDER": {
      if (state.phase !== "orderChoice") return state;
      if (action.playerId !== state.hostId) return state;

      return {
        ...state,
        orderMode: action.mode,
        phase: "preparation",
        targetPlayerId: null,
        proposals: [],
        submittedForTurn: [],
        demaskingVotes: {},
        demaskingResult: null,
        demaskingTie: false,
      };
    }

    case "SUBMIT_PROPOSALS": {
      if (state.phase !== "preparation") return state;
      if (state.submittedForTurn.includes(action.playerId)) return state;

      const targetId = state.currentPlayerId;

      // The target NEVER writes a proposal for their own turn.
      if (action.playerId === targetId) return state;

      if (!action.actionText.trim() || !action.truthText.trim()) return state;

      const proposals = [
        ...state.proposals,
        {
          id: uid(),
          type: "action" as const,
          text: action.actionText.trim(),
          authorId: action.playerId,
        },
        {
          id: uid(),
          type: "truth" as const,
          text: action.truthText.trim(),
          authorId: action.playerId,
        },
      ];

      const submittedForTurn = [
        ...state.submittedForTurn,
        action.playerId,
      ];

      const required = state.players.filter((p) => p.id !== targetId).length;
      const complete = submittedForTurn.length === required;

      return {
        ...state,
        proposals,
        submittedForTurn,
        phase: complete ? "play" : "preparation",
      };
    }

    case "SUBMIT_TARGET_PROPOSAL": {
      if (state.phase !== "preparation") return state;
      if (state.gameMode !== "target") return state;
      if (action.playerId !== state.currentPlayerId) return state;
      if (state.submittedForTurn.includes(action.playerId)) return state;
      if (action.targetPlayerId === action.playerId) return state;
      if (!state.players.some((p) => p.id === action.targetPlayerId)) return state;
      if (!action.text.trim()) return state;

      const proposal: Proposal = {
        id: uid(),
        type: action.typeChoice,
        text: action.text.trim(),
        authorId: action.playerId,
      };

      return {
        ...state,
        proposals: [proposal],
        submittedForTurn: [action.playerId],
        targetPlayerId: action.targetPlayerId,
        selectedProposalId: null,
        revealed: false,
        phase: "play",
      };
    }

    case "SELECT_PROPOSAL": {
      if (state.phase !== "play") return state;
      const expectedPlayer = state.gameMode === "target"
        ? state.targetPlayerId
        : state.currentPlayerId;
      if (action.playerId !== expectedPlayer) return state;

      const exists = state.proposals.some(
        (p) => p.id === action.proposalId
      );
      if (!exists) return state;

      // Selecting a card only selects it.
      // Its content stays hidden until REVEAL_PROPOSAL.
      return {
        ...state,
        selectedProposalId: action.proposalId,
        revealed: false,
        phase: "reveal",
      };
    }

    case "REVEAL_PROPOSAL": {
      if (state.phase !== "reveal") return state;
      const expectedPlayer = state.gameMode === "target"
        ? state.targetPlayerId
        : state.currentPlayerId;
      if (action.playerId !== expectedPlayer) return state;
      if (!state.selectedProposalId) return state;

      return { ...state, revealed: true };
    }

    case "NEXT_TURN": {
      if (state.phase !== "reveal") return state;
      const expectedPlayer = state.gameMode === "target"
        ? state.targetPlayerId
        : state.currentPlayerId;
      if (action.playerId !== expectedPlayer) return state;
      if (!state.revealed) return state;

      // En Mode Intense, la tentative de démasquage se déclenche
      // immédiatement après l'exécution/la réponse.
      if (state.gameMode === "intense" && state.demaskingEnabled) {
        return {
          ...state,
          phase: "demaskingVote",
          demaskingVotes: {},
          demaskingResult: null,
          demaskingTie: false,
        };
      }

      let nextIndex = 0;
      let nextId = state.players[0]?.id ?? state.currentPlayerId;

      if (state.orderMode === "order") {
        nextIndex = (state.turnIndex + 1) % state.players.length;
        nextId = state.players[nextIndex]?.id ?? state.currentPlayerId;
      } else {
        const next = randomNextPlayer(state);
        nextIndex = next.index;
        nextId = next.id;
      }

      return {
        ...state,
        phase: "preparation",
        turnIndex: nextIndex,
        currentPlayerId: nextId,
        targetPlayerId: null,
        proposals: [],
        submittedForTurn: [],
        selectedProposalId: null,
        revealed: false,
        demaskingVotes: {},
        demaskingResult: null,
        demaskingTie: false,
        roundIdentity:
          state.identityMode === "hidden"
            ? "hidden"
            : "reveal",
      };
    }

    case "SUBMIT_DEMASKING_VOTE": {
      if (state.phase !== "demaskingVote") return state;
      if (state.gameMode !== "intense" || !state.demaskingEnabled) return state;
      if (!state.players.some((p) => p.id === action.playerId)) return state;
      if (state.demaskingVotes[action.playerId]) return state;

      const authors = Array.from(new Set(
        state.proposals
          .filter((p) => p.authorId !== state.currentPlayerId)
          .map((p) => p.authorId)
      ));
      if (!authors.includes(action.authorId)) return state;

      const votes = { ...state.demaskingVotes, [action.playerId]: action.authorId };
      if (Object.keys(votes).length < state.players.length) {
        return { ...state, demaskingVotes: votes };
      }

      const counts: Record<string, number> = {};
      authors.forEach((id) => { counts[id] = 0; });
      Object.values(votes).forEach((authorId) => { counts[authorId] = (counts[authorId] ?? 0) + 1; });
      const maxVotes = Math.max(...Object.values(counts));
      const winners = authors.filter((id) => counts[id] === maxVotes);

      return {
        ...state,
        demaskingVotes: votes,
        demaskingResult: winners.length === 1 ? winners[0] : null,
        demaskingTie: winners.length !== 1,
        phase: "demaskingResult",
      };
    }

    case "CONTINUE_AFTER_DEMASKING": {
      if (state.phase !== "demaskingResult") return state;
      if (action.playerId !== state.currentPlayerId) return state;

      let nextIndex = 0;
      let nextId = state.players[0]?.id ?? state.currentPlayerId;
      if (state.orderMode === "order") {
        nextIndex = (state.turnIndex + 1) % state.players.length;
        nextId = state.players[nextIndex]?.id ?? state.currentPlayerId;
      } else {
        const next = randomNextPlayer(state);
        nextIndex = next.index;
        nextId = next.id;
      }

      return {
        ...state,
        phase: "preparation",
        turnIndex: nextIndex,
        currentPlayerId: nextId,
        targetPlayerId: null,
        proposals: [],
        submittedForTurn: [],
        selectedProposalId: null,
        revealed: false,
        demaskingVotes: {},
        demaskingResult: null,
        demaskingTie: false,
        roundIdentity: state.identityMode === "hidden" ? "hidden" : "reveal",
      };
    }

    case "NEXT_ROUND": {
      if (state.phase !== "roundEnd") return state;
      if (action.playerId !== state.hostId) return state;

      if (state.round >= state.maxRounds) {
        return { ...state, phase: "gameEnd" };
      }

      const nextIndex =
        state.orderMode === "order"
          ? (state.turnIndex + 1) % state.players.length
          : randomNextPlayer(state).index;

      return {
        ...state,
        round: state.round + 1,
        phase: "preparation",
        turnIndex: nextIndex,
        currentPlayerId: state.players[nextIndex]?.id ?? state.currentPlayerId,
        targetPlayerId: null,
        proposals: [],
        submittedForTurn: [],
        selectedProposalId: null,
        revealed: false,
      };
    }

    case "LEAVE": {
      const activeTurn =
        state.phase === "preparation" ||
        state.phase === "play" ||
        state.phase === "reveal" ||
        state.phase === "demaskingVote" ||
        state.phase === "demaskingResult";

      const cannotLeave =
        activeTurn &&
        (action.playerId === state.currentPlayerId ||
          action.playerId === state.targetPlayerId);

      if (cannotLeave) return state;

      return removePlayerFromState(state, action.playerId);
    }

    case "ADD_PLAYER_DIRECT": {
      if (action.playerId !== state.hostId) return state;
      const cleanName = action.name.trim();
      if (!cleanName || state.players.length >= 20) return state;
      const id = uid();
      const player: Player = { id, name: cleanName, avatar: "🌟" };
      return { ...state, players: [...state.players, player], score: { ...state.score, [id]: 0 } };
    }

    case "DISCONNECT_PLAYER":
      // Une déconnexion réseau est traitée comme un départ forcé.
      // Le Maître de jeu restant est le seul à valider cette suppression.
      if (action.playerId !== state.hostId) return state;
      if (action.targetPlayerId === state.hostId) return state;
      return removePlayerFromState(state, action.targetPlayerId);

    case "REMOVE_PLAYER":
      if (action.playerId !== state.hostId) return state;
      if (action.targetPlayerId === state.hostId) return state;
      return removePlayerFromState(state, action.targetPlayerId);

    default:
      return state;
  }
}

const TEXT: Record<Lang, Record<string, string>> = {
  fr: {
    subtitle: "ACTION OU VÉRITÉ",
    accountOptional: "Joue directement, sans compte.",
    guest: "JOUER SANS COMPTE",
    create: "CRÉER UNE PARTIE",
    join: "REJOINDRE",
    space: "RÈGLES DU JEU",
    language: "LANGUE",
    name: "Ton prénom ou pseudo",
    code: "Code de partie",
    back: "Retour",
    createNow: "Créer la partie",
    joinNow: "Rejoindre la partie",
    lobby: "LOBBY",
    host: "Maître de jeu",
    start: "Lancer la configuration",
    need3: "Le Mode Intense nécessite 3 joueurs. Le Mode Cible peut jouer à 2.",
    pendingJoin: "Demandes pour rejoindre",
    accept: "Accepter",
    reject: "Refuser",
    removePlayer: "Retirer",
    leaveGame: "Quitter la partie",
    leaveConfirm: "Quitter la partie ?",
    leaveConfirmText: "La partie continue si le nombre minimum de joueurs est conservé.",
    stay: "Rester",
    waitingApproval: "Demande envoyée. Le Maître de jeu doit t'accepter.",
    removed: "Tu as été retiré de la partie.",
    settings: "RÈGLES DE LA PARTIE",
    closeSettings: "Fermer",
    hostOnly: "Seul le Maître de jeu peut modifier ces réglages.",
    gameMode: "MODE DE JEU",
    targetMode: "🎯 MODE CIBLE",
    targetModeHelp: "À partir de 2 joueurs. Le joueur actif choisit directement une cible et lui propose une seule Action ou Vérité.",
    intenseMode: "🔥 MODE INTENSE",
    intenseModeHelp: "À partir de 3 joueurs. La cible reçoit plusieurs propositions et choisit une Action ou une Vérité sans voir le contenu.",
    chooseTarget: "Choisis une cible",
    proposalType: "Type de proposition",
    writeProposal: "Écris ta proposition",
    confirmTarget: "Valider la cible",
    identities: "Identités",
    revealed: "👁️ Identités révélées",
    hidden: "🙈 Identités cachées",
    order: "ORDRE",
    disorder: "DÉSORDRE",
    orderHelp: "Ordre : la séquence des joueurs reste fixe.",
    disorderHelp: "Désordre : l'application choisit aléatoirement le prochain joueur.",
    prepare: "PRÉPARATION",
    prepareHelp: "Écris une Action et une Vérité pour la cible. Rien n'est conservé après ce tour.",
    target: "CIBLE",
    action: "🔥 ACTION",
    truth: "💬 VÉRITÉ",
    send: "Valider",
    waiting: "En attente des autres joueurs…",
    choose: "CHOISIS UNE CARTE",
    chooseHelp: "Tu vois uniquement le dos de la carte : ACTION ou VÉRITÉ. Le contenu est caché.",
    reveal: "RETOURNER LA CARTE",
    revealedContent: "CARTE RÉVÉLÉE",
    done: "J'AI FAIT / RÉPONDU",
    use: "UTILISER",
    private: "Message discret",
    sendMessage: "Envoyer",
    roundEnd: "FIN DE MANCHE",
    next: "MANCHE SUIVANTE",
    gameEnd: "FIN DE PARTIE",
    avatar: "Choisis ton avatar",
    skip: "Passer",
    saveAvatar: "Continuer",
    noKey: "Configure ta clé publique Supabase pour utiliser le multijoueur.",
  },
  en: {
    subtitle: "TRUTH OR DARE",
    accountOptional: "Play directly, no account needed.",
    guest: "PLAY WITHOUT ACCOUNT",
    create: "CREATE GAME",
    join: "JOIN",
    space: "MY SPACE",
    language: "LANGUAGE",
    name: "Your name or nickname",
    code: "Game code",
    back: "Back",
    createNow: "Create game",
    joinNow: "Join game",
    lobby: "LOBBY",
    host: "Game master",
    start: "Start setup",
    need3: "Intense Mode requires 3 players. Target Mode can be played with 2.",
    pendingJoin: "Join requests", accept: "Accept", reject: "Reject", removePlayer: "Remove", leaveGame: "Leave game", leaveConfirm: "Leave the game?", leaveConfirmText: "The game continues if the minimum number of players is maintained.", stay: "Stay", waitingApproval: "Request sent. The Game Master must accept you.", removed: "You were removed from the game.",
    settings: "GAME RULES",
    closeSettings: "Close",
    hostOnly: "Only the Game Master can change these settings.",
    gameMode: "GAME MODE",
    targetMode: "🎯 TARGET MODE",
    targetModeHelp: "From 2 players. The active player directly chooses a target and sends one Dare or Truth.",
    intenseMode: "🔥 INTENSE MODE",
    intenseModeHelp: "From 3 players. The target receives several proposals and chooses Dare or Truth without seeing the content.",
    chooseTarget: "Choose a target",
    proposalType: "Proposal type",
    writeProposal: "Write your proposal",
    confirmTarget: "Confirm target",
    identities: "Identities",
    revealed: "👁️ Identities revealed",
    hidden: "🙈 Identities hidden",
    order: "ORDER",
    disorder: "DISORDER",
    orderHelp: "Order: the player sequence stays fixed.",
    disorderHelp: "Disorder: the app randomly chooses the next player.",
    prepare: "PREPARATION",
    prepareHelp: "Write one Dare and one Truth for the target. Nothing is saved after this turn.",
    target: "TARGET",
    action: "🔥 DARE",
    truth: "💬 TRUTH",
    send: "Submit",
    waiting: "Waiting for the other players…",
    choose: "CHOOSE A CARD",
    chooseHelp: "You only see the card back: DARE or TRUTH. The content is hidden.",
    reveal: "FLIP CARD",
    revealedContent: "REVEALED CARD",
    done: "DONE",
    use: "USE",
    private: "Private message",
    sendMessage: "Send",
    roundEnd: "ROUND END",
    next: "NEXT ROUND",
    gameEnd: "GAME OVER",
    avatar: "Choose your avatar",
    skip: "Skip",
    saveAvatar: "Continue",
    noKey: "Configure your public Supabase key to use multiplayer.",
  },
  es: {
    subtitle: "VERDAD O RETO",
    accountOptional: "Juega directamente, sin cuenta.",
    guest: "JUGAR SIN CUENTA",
    create: "CREAR PARTIDA",
    join: "UNIRSE",
    space: "MI ESPACIO",
    language: "IDIOMA",
    name: "Tu nombre o apodo",
    code: "Código",
    back: "Volver",
    createNow: "Crear partida",
    joinNow: "Unirse",
    settings: "REGLAS",
    gameMode: "MODO DE JUEGO",
    targetMode: "🎯 MODO OBJETIVO",
    targetModeHelp: "Desde 2 jugadores. El jugador activo elige directamente un objetivo y le envía un Reto o Verdad.",
    intenseMode: "🔥 MODO INTENSO",
    intenseModeHelp: "Desde 3 jugadores. El objetivo recibe varias propuestas y elige Reto o Verdad sin ver el contenido.",
    chooseTarget: "Elige un objetivo",
    proposalType: "Tipo de propuesta",
    writeProposal: "Escribe tu propuesta",
    confirmTarget: "Confirmar objetivo",
    identities: "Identidades",
    revealed: "👁️ Identidades visibles",
    hidden: "🙈 Identidades ocultas",
    order: "ORDEN",
    disorder: "DESORDEN",
    orderHelp: "El orden permanece fijo.",
    disorderHelp: "La app elige al azar al siguiente jugador.",
    prepare: "PREPARACIÓN",
    prepareHelp: "Escribe un Reto y una Verdad para el objetivo. Nada se guarda.",
    target: "OBJETIVO",
    action: "🔥 RETO",
    truth: "💬 VERDAD",
    send: "Validar",
    waiting: "Esperando…",
    choose: "ELIGE UNA CARTA",
    chooseHelp: "Solo ves el reverso: RETO o VERDAD. El contenido está oculto.",
    reveal: "DAR LA VUELTA",
    revealedContent: "CARTA REVELADA",
    done: "HECHO",
    use: "USAR",
    private: "Mensaje privado",
    sendMessage: "Enviar",
    roundEnd: "FIN DE RONDA",
    next: "SIGUIENTE RONDA",
    gameEnd: "FIN DE PARTIDA",
    avatar: "Elige tu avatar",
    skip: "Saltar",
    saveAvatar: "Continuar",
    noKey: "Configura tu clave pública de Supabase para el multijugador.",
    need3: "El Modo Intenso necesita 3 jugadores. El Modo Objetivo puede jugarse con 2.",
    pendingJoin: "Solicitudes para unirse", accept: "Aceptar", reject: "Rechazar", removePlayer: "Expulsar", leaveGame: "Salir de la partida", leaveConfirm: "¿Salir de la partida?", leaveConfirmText: "La partida continúa si se mantiene el mínimo de jugadores.", stay: "Quedarse", waitingApproval: "Solicitud enviada. El Maestro debe aceptarte.", removed: "Has sido expulsado de la partida.",
  },
  it: {
    subtitle: "OBBLIGO O VERITÀ",
    accountOptional: "Gioca direttamente, senza account.",
    guest: "GIOCA SENZA ACCOUNT",
    create: "CREA PARTITA",
    join: "UNISCITI",
    space: "IL MIO SPAZIO",
    language: "LINGUA",
    name: "Nome o nickname",
    code: "Codice",
    back: "Indietro",
    createNow: "Crea partita",
    joinNow: "Unisciti",
    lobby: "LOBBY",
    host: "Maestro",
    start: "Avvia configurazione",
    need3: "Il Mode Intense richiede 3 giocatori. Il Mode Cible può essere giocato in 2.",
    pendingJoin: "Richieste di ingresso", accept: "Accetta", reject: "Rifiuta", removePlayer: "Rimuovi", leaveGame: "Lascia la partita", leaveConfirm: "Lasciare la partita?", leaveConfirmText: "La partita continua se viene mantenuto il numero minimo di giocatori.", stay: "Resta", waitingApproval: "Richiesta inviata. Il Maestro deve accettarti.", removed: "Sei stato rimosso dalla partita.",
    settings: "REGOLE",
    gameMode: "MODALITÀ DI GIOCO",
    targetMode: "🎯 MODE CIBLE",
    targetModeHelp: "Da 2 giocatori. Il giocatore attivo sceglie direttamente un bersaglio e invia una sola Azione o Verità.",
    intenseMode: "🔥 MODE INTENSE",
    intenseModeHelp: "Il bersaglio riceve diverse proposte e sceglie Azione o Verità senza vedere il contenuto.",
    chooseTarget: "Scegli un bersaglio",
    proposalType: "Tipo di proposta",
    writeProposal: "Scrivi la proposta",
    confirmTarget: "Conferma bersaglio",
    identities: "Identità",
    revealed: "👁️ Identità rivelate",
    hidden: "🙈 Identità nascoste",
    order: "ORDINE",
    disorder: "DISORDINE",
    orderHelp: "La sequenza resta fissa.",
    disorderHelp: "L'app sceglie casualmente il prossimo giocatore.",
    prepare: "PREPARAZIONE",
    prepareHelp: "Scrivi un Obbligo e una Verità per il bersaglio. Nulla viene salvato.",
    target: "BERSAGLIO",
    action: "🔥 OBBLIGO",
    truth: "💬 VERITÀ",
    send: "Conferma",
    waiting: "In attesa…",
    choose: "SCEGLI UNA CARTA",
    chooseHelp: "Vedi solo il retro: OBBLIGO o VERITÀ. Il contenuto è nascosto.",
    reveal: "GIRA LA CARTA",
    revealedContent: "CARTA RIVELATA",
    done: "FATTO",
    use: "USA",
    private: "Messaggio privato",
    sendMessage: "Invia",
    roundEnd: "FINE ROUND",
    next: "ROUND SUCCESSIVO",
    gameEnd: "FINE PARTITA",
    avatar: "Scegli avatar",
    skip: "Salta",
    saveAvatar: "Continua",
    noKey: "Configura la chiave pubblica Supabase per il multigiocatore.",
  },
  de: {
    subtitle: "WAHRHEIT ODER PFLICHT",
    accountOptional: "Spiele direkt, ohne Konto.",
    guest: "OHNE KONTO SPIELEN",
    create: "SPIEL ERSTELLEN",
    join: "BEITRETEN",
    space: "MEIN BEREICH",
    language: "SPRACHE",
    name: "Name oder Spitzname",
    code: "Spielcode",
    back: "Zurück",
    createNow: "Spiel erstellen",
    joinNow: "Beitreten",
    lobby: "LOBBY",
    host: "Spielleiter",
    start: "Setup starten",
    need3: "Der Intensivmodus braucht 3 Spieler. Der Zielmodus kann mit 2 gespielt werden.",
    pendingJoin: "Beitrittsanfragen", accept: "Annehmen", reject: "Ablehnen", removePlayer: "Entfernen", leaveGame: "Spiel verlassen", leaveConfirm: "Spiel verlassen?", leaveConfirmText: "Das Spiel läuft weiter, wenn die Mindestspielerzahl erhalten bleibt.", stay: "Bleiben", waitingApproval: "Anfrage gesendet. Der Spielleiter muss dich annehmen.", removed: "Du wurdest aus dem Spiel entfernt.",
    settings: "REGELN",
    gameMode: "SPIELMODUS",
    targetMode: "🎯 ZIELMODUS",
    targetModeHelp: "Der aktive Spieler wählt direkt ein Ziel und sendet eine Pflicht oder Wahrheit.",
    intenseMode: "🔥 INTENSIVMODUS",
    intenseModeHelp: "Das Ziel erhält mehrere Vorschläge und wählt Pflicht oder Wahrheit, ohne den Inhalt zu sehen.",
    chooseTarget: "Ziel wählen",
    proposalType: "Vorschlagstyp",
    writeProposal: "Vorschlag schreiben",
    confirmTarget: "Ziel bestätigen",
    identities: "Identitäten",
    revealed: "👁️ Identitäten sichtbar",
    hidden: "🙈 Identitäten verborgen",
    order: "ORDNUNG",
    disorder: "CHAOS",
    orderHelp: "Die Reihenfolge bleibt fest.",
    disorderHelp: "Die App wählt den nächsten Spieler zufällig.",
    prepare: "VORBEREITUNG",
    prepareHelp: "Schreibe eine Pflicht und eine Wahrheit für das Ziel. Nichts wird gespeichert.",
    target: "ZIEL",
    action: "🔥 PFLICHT",
    truth: "💬 WAHRHEIT",
    send: "Bestätigen",
    waiting: "Warten…",
    choose: "KARTE WÄHLEN",
    chooseHelp: "Du siehst nur die Rückseite: PFLICHT oder WAHRHEIT. Der Inhalt ist verborgen.",
    reveal: "KARTE UMDREHEN",
    revealedContent: "KARTE AUFGEDECKT",
    done: "FERTIG",
    use: "BENUTZEN",
    private: "Private Nachricht",
    sendMessage: "Senden",
    roundEnd: "RUNDENENDE",
    next: "NÄCHSTE RUNDE",
    gameEnd: "SPIELENDE",
    avatar: "Avatar wählen",
    skip: "Überspringen",
    saveAvatar: "Weiter",
    noKey: "Konfiguriere deinen öffentlichen Supabase-Schlüssel für Multiplayer.",
  },
};

const LANGUAGES: { code: Lang; label: string }[] = [
  { code: "fr", label: "Français 🇫🇷" },
  { code: "en", label: "English 🇬🇧" },
  { code: "es", label: "Español 🇪🇸" },
  { code: "it", label: "Italiano 🇮🇹" },
  { code: "de", label: "Deutsch 🇩🇪" },
];

const AVATARS = ["🌸", "🔥", "⚡", "🐯", "🦊", "🐼", "🐸", "🦄"];

function Button({
  label,
  onPress,
  variant = "primary",
  disabled = false,
}: {
  label: string;
  onPress: () => void;
  variant?: "primary" | "pink" | "yellow" | "outline";
  disabled?: boolean;
}) {
  return (
    <TouchableOpacity
      activeOpacity={0.85}
      disabled={disabled}
      onPress={onPress}
      style={[
        styles.button,
        variant === "pink" && styles.buttonPink,
        variant === "yellow" && styles.buttonYellow,
        variant === "outline" && styles.buttonOutline,
        disabled && styles.buttonDisabled,
      ]}
    >
      <Text
        style={[
          styles.buttonText,
          variant === "outline" && styles.buttonOutlineText,
        ]}
      >
        {label}
      </Text>
    </TouchableOpacity>
  );
}

function CardBack({
  type,
  index,
  onPress,
  actionBackground,
  actionBorder,
  truthBackground,
  truthBorder,
}: {
  key?: string;
  type: "action" | "truth";
  index: number;
  onPress: () => void;
  actionBackground?: string;
  actionBorder?: string;
  truthBackground?: string;
  truthBorder?: string;
}) {
  return (
    <TouchableOpacity
      activeOpacity={0.9}
      onPress={onPress}
      style={[
        styles.cardBack,
        type === "action"
          ? [
              styles.actionBack,
              actionBackground ? { backgroundColor: actionBackground } : null,
              actionBorder ? { borderColor: actionBorder } : null,
            ]
          : [
              styles.truthBack,
              truthBackground ? { backgroundColor: truthBackground } : null,
              truthBorder ? { borderColor: truthBorder } : null,
            ],
      ]}
    >
      <Text style={styles.cardIndex}>{index + 1}</Text>
      <Text style={styles.cardBackEmoji}>
        {type === "action" ? "🔥" : "💬"}
      </Text>
      <Text style={styles.cardBackType}>
        {type === "action" ? "ACTION" : "VÉRITÉ"}
      </Text>
      <Text style={styles.cardBackHidden}>CONTENU CACHÉ</Text>
    </TouchableOpacity>
  );
}

export default function App() {
  const [lang, setLang] = useState<Lang>("fr");
  const t = TEXT[lang];

  const [screen, setScreen] = useState<Screen>("welcome");
  const [name, setName] = useState("");
  const [avatar, setAvatar] = useState("🌸");
  const [avatarChoice, setAvatarChoice] = useState("🌸");
  const [code, setCode] = useState("");
  const [error, setError] = useState("");

  const [game, setGame] = useState<GameState | null>(null);
  const gameRef = useRef<GameState | null>(null);
  const channelRef = useRef<RealtimeChannel | null>(null);
  const privateChannelsRef = useRef<Map<string, RealtimeChannel>>(new Map());
  const isHostRef = useRef(false);
  const myIdRef = useRef(uid());
  const presenceRecoveryTimerRef = useRef<ReturnType<typeof setTimeout> | null>(null);
  const presenceMissesRef = useRef<Map<string, number>>(new Map());
  const processedActionIdsRef = useRef<Set<string>>(new Set());
  const reconnectTimerRef = useRef<ReturnType<typeof setTimeout> | null>(null);
  const [unreadPrivateCount, setUnreadPrivateCount] = useState(0);
  const [privateAlert, setPrivateAlert] = useState<{ fromName: string; text: string } | null>(null);

  const [actionText, setActionText] = useState("");
  const [truthText, setTruthText] = useState("");
  const [privateTarget, setPrivateTarget] = useState<string | null>(null);
  const [privateText, setPrivateText] = useState("");
  const [privateMessages, setPrivateMessages] = useState<{ id: string; fromId: string; toId: string; text: string }[]>([]);
  const [addPlayerName, setAddPlayerName] = useState("");
  const [targetChoice, setTargetChoice] = useState<string | null>(null);
  const [targetType, setTargetType] = useState<"action" | "truth">("truth");
  const [targetProposalText, setTargetProposalText] = useState("");
  const [showInGameSettings, setShowInGameSettings] = useState(false);
  const [showPrivateComposer, setShowPrivateComposer] = useState(false);
  const intensePulse = useRef(new Animated.Value(0)).current;

  useEffect(() => {
    if (game?.gameMode !== "intense") {
      intensePulse.stopAnimation();
      intensePulse.setValue(0);
      return;
    }
    const loop = Animated.loop(
      Animated.sequence([
        Animated.timing(intensePulse, { toValue: 1, duration: 1500, easing: Easing.inOut(Easing.ease), useNativeDriver: true }),
        Animated.timing(intensePulse, { toValue: 0, duration: 1500, easing: Easing.inOut(Easing.ease), useNativeDriver: true }),
      ])
    );
    loop.start();
    return () => loop.stop();
  }, [game?.gameMode]);

  const myId = myIdRef.current;

  const liveTheme = useMemo(
    () =>
      game?.gameMode === "intense"
        ? {
            background: "#FFF3F1",
            surface: "#FFFFFF",
            primary: "#D7193F",
            primarySoft: "#FFE1E7",
            secondary: "#FF6B35",
            accent: "#FFB347",
            text: "#35151D",
            muted: "#7B5660",
            hero: "#C9143A",
            heroSoft: "#FFE7EB",
            cardAction: "#FFE0E5",
            cardTruth: "#FFF0E8",
            border: "#F0A9B5",
          }
        : {
            background: "#FFF7FC",
            surface: "#FFFFFF",
            primary: "#C83D9B",
            primarySoft: "#F9E5F4",
            secondary: "#7C4DFF",
            accent: "#FFE36E",
            text: "#2D2140",
            muted: "#75687F",
            hero: "#9A5DE0",
            heroSoft: "#F0E6FF",
            cardAction: "#FFF0F5",
            cardTruth: "#F0E9FF",
            border: "#E8D6EA",
          },
    [game?.gameMode]
  );

  const currentPlayer = useMemo(
    () => game?.players.find((p) => p.id === game.currentPlayerId),
    [game]
  );

  const isMyTurn = game?.currentPlayerId === myId;

  const selectedProposal = useMemo(
    () => game?.proposals.find((p) => p.id === game.selectedProposalId),
    [game]
  );

  useEffect(() => {
    if (!game || isHostRef.current || screen !== "game") return;
    const isMember = game.players.some((p) => p.id === myId);
    const isPending = game.joinRequests.some((p) => p.id === myId);
    if (!isMember && !isPending && game.phase !== "lobby") {
      Alert.alert("AVOU", t.removed);
      closeChannel();
      gameRef.current = null;
      setGame(null);
      setScreen("join");
    }
  }, [game, screen, myId, t.removed]);

  useEffect(() => {
    if (!game || screen !== "game" || !supabase) return;
    refreshPrivateChannels(game.code, game.players);
  }, [game?.code, game?.players, screen]);

  useEffect(() => {
    if (!game) return;
    // Le rôle d'hôte est dérivé de GAME_STATE. Après un transfert,
    // le nouveau Maître obtient immédiatement toutes les permissions.
    isHostRef.current = game.hostId === myId;
  }, [game?.hostId, myId]);

  function schedulePresenceReconciliation() {
    if (presenceRecoveryTimerRef.current) {
      clearTimeout(presenceRecoveryTimerRef.current);
    }

    presenceRecoveryTimerRef.current = setTimeout(() => {
      const current = gameRef.current;
      const channel = channelRef.current;
      if (!current || !channel) return;

      const presenceState = channel.presenceState();
      const presentIds = new Set(Object.keys(presenceState));

      // On tolère les micro-coupures réseau avant de retirer un joueur.
      for (const player of current.players) {
        if (player.id === myId) continue;
        if (presentIds.has(player.id)) {
          presenceMissesRef.current.delete(player.id);
        } else {
          const misses = (presenceMissesRef.current.get(player.id) ?? 0) + 1;
          presenceMissesRef.current.set(player.id, misses);
        }
      }

      const missingHost = current.hostId && !presentIds.has(current.hostId);
      const hostMisses = current.hostId
        ? (presenceMissesRef.current.get(current.hostId) ?? 0)
        : 0;

      // Il faut deux contrôles successifs avant de considérer l'hôte réellement parti.
      if (missingHost && hostMisses >= 2) {
        const presentCandidates = current.players.filter(
          (p) => p.id !== current.hostId && presentIds.has(p.id)
        );
        const newHostId = deterministicHostId(current, current.hostId, presentCandidates);

        // Un seul appareil est autorisé à finaliser le transfert.
        if (newHostId === myId) {
          const next = removePlayerFromState(current, current.hostId, newHostId);
          const versioned = { ...next, stateVersion: current.stateVersion + 1 };
          setLocalState(versioned);
          broadcastState(versioned);
        }
        return;
      }

      // Pour un autre joueur réellement déconnecté, seul l'hôte actuel nettoie la salle.
      if (isHostRef.current) {
        const disconnected = current.players.find(
          (p) => p.id !== myId &&
            !presentIds.has(p.id) &&
            (presenceMissesRef.current.get(p.id) ?? 0) >= 2
        );
        if (disconnected) {
          hostDispatch({
            type: "DISCONNECT_PLAYER",
            playerId: myId,
            targetPlayerId: disconnected.id,
          });
        }
      }
    }, 2200);
  }

  function setLocalState(next: GameState) {
    gameRef.current = next;
    isHostRef.current = next.hostId === myId;
    setGame(next);
  }

  function broadcastState(next: GameState) {
    channelRef.current?.send({
      type: "broadcast",
      event: "GAME_STATE",
      payload: next,
    });
  }

  function broadcastSyncRequest() {
    channelRef.current?.send({
      type: "broadcast",
      event: "SYNC_REQUEST",
      payload: { requesterId: myId },
    });
  }

  function hostDispatch(action: Action) {
    const current = gameRef.current;
    if (!current || current.hostId !== myId) return;

    const reduced = reducer(current, action);
    if (reduced === current) return;

    const next = { ...reduced, stateVersion: current.stateVersion + 1 };
    setLocalState(next);
    broadcastState(next);
  }

  function dispatch(action: Action) {
    if (isHostRef.current) {
      hostDispatch(action);
      return;
    }

    const actionId = uid();
    channelRef.current?.send({
      type: "broadcast",
      event: "GAME_ACTION",
      payload: { actionId, action },
    });
  }

  function privateChannelKey(gameCode: string, a: string, b: string) {
    return `avou-${gameCode}-dm-${[a, b].sort().join("-")}`;
  }

  function closePrivateChannels() {
    for (const channel of privateChannelsRef.current.values()) {
      supabase?.removeChannel(channel);
    }
    privateChannelsRef.current.clear();
  }

  function refreshPrivateChannels(gameCode: string, players: Player[]) {
    if (!supabase) return;

    const others = players.filter((p) => p.id !== myId);
    const wanted = new Set(others.map((p) => privateChannelKey(gameCode, myId, p.id)));

    for (const [key, channel] of privateChannelsRef.current.entries()) {
      if (!wanted.has(key)) {
        supabase.removeChannel(channel);
        privateChannelsRef.current.delete(key);
      }
    }

    for (const other of others) {
      const key = privateChannelKey(gameCode, myId, other.id);
      if (privateChannelsRef.current.has(key)) continue;

      const channel = supabase.channel(key, {
        config: { broadcast: { self: false } },
      });

      channel.on("broadcast", { event: "PRIVATE_MESSAGE" }, (data: any) => {
        const message = data?.payload;
        if (!message) return;
        if (message.toId !== myId && message.fromId !== myId) return;
        setPrivateMessages((prev) =>
          prev.some((m) => m.id === message.id) ? prev : [...prev, message]
        );

        if (message.toId === myId) {
          const senderName = gameRef.current?.players.find((p) => p.id === message.fromId)?.name ?? "Joueur";
          setUnreadPrivateCount((count) => count + 1);
          setPrivateAlert({ fromName: senderName, text: message.text });
          Alert.alert("Nouveau message privé", `${senderName} : ${message.text}`, [
            {
              text: "Ouvrir",
              onPress: () => {
                setPrivateTarget(message.fromId);
                setShowPrivateComposer(true);
                setUnreadPrivateCount((count) => Math.max(0, count - 1));
              },
            },
            { text: "Fermer", style: "cancel" },
          ]);
        }
      });

      channel.subscribe();
      privateChannelsRef.current.set(key, channel);
    }
  }

  function sendPrivateMessage(toId: string, textValue: string) {
    const clean = textValue.trim();
    const current = gameRef.current;
    if (!clean || !supabase || !current) return;
    if (toId === myId || !current.players.some((p) => p.id === toId)) return;

    const key = privateChannelKey(current.code, myId, toId);
    const channel = privateChannelsRef.current.get(key);
    if (!channel) {
      refreshPrivateChannels(current.code, current.players);
      return;
    }

    const message = { id: uid(), fromId: myId, toId, text: clean };
    setPrivateMessages((prev) => [...prev, message]);
    channel.send({ type: "broadcast", event: "PRIVATE_MESSAGE", payload: message });
  }

  function closeChannel() {
    if (presenceRecoveryTimerRef.current) {
      clearTimeout(presenceRecoveryTimerRef.current);
      presenceRecoveryTimerRef.current = null;
    }
    closePrivateChannels();
    presenceMissesRef.current.clear();
    if (reconnectTimerRef.current) {
      clearTimeout(reconnectTimerRef.current);
      reconnectTimerRef.current = null;
    }
    if (channelRef.current) {
      supabase?.removeChannel(channelRef.current);
      channelRef.current = null;
    }
    setPrivateMessages([]);
    setUnreadPrivateCount(0);
    setPrivateAlert(null);
  }

  function setupChannel(gameCode: string, onConnected: () => void) {
    closeChannel();

    if (!supabase) {
      setError(t.noKey);
      return;
    }

    const channel = supabase.channel(`avou-${gameCode}`, {
      config: {
        broadcast: { self: false },
        presence: { key: myId },
      },
    });

    channel.on("broadcast", { event: "GAME_STATE" }, (data: any) => {
      if (!data?.payload) return;
      const next = { ...(data.payload as GameState), stateVersion: Number((data.payload as GameState).stateVersion ?? 1) };
      const current = gameRef.current;

      // Un ancien paquet réseau ne doit jamais faire revenir la partie en arrière.
      if (current && Number(next.stateVersion ?? 0) < Number(current.stateVersion ?? 0)) return;

      // Un appareil qui n'est plus membre ne reprend pas l'état public comme s'il
      // avait été accepté. L'écran d'attente est géré par l'état players/joinRequests.
      if (!isHostRef.current) {
        setLocalState(next);
        refreshPrivateChannels(next.code, next.players);
      }
    });

    channel.on("broadcast", { event: "GAME_ACTION" }, (data: any) => {
      if (!isHostRef.current || !data?.payload) return;
      const packet = data.payload;
      const actionId = typeof packet?.actionId === "string" ? packet.actionId : uid();
      const action = packet?.action ?? packet;
      if (processedActionIdsRef.current.has(actionId)) return;
      processedActionIdsRef.current.add(actionId);
      if (processedActionIdsRef.current.size > 500) {
        const first = processedActionIdsRef.current.values().next().value;
        if (first) processedActionIdsRef.current.delete(first);
      }
      hostDispatch(action as Action);
    });

    channel.on("broadcast", { event: "SYNC_REQUEST" }, () => {
      if (isHostRef.current && gameRef.current) {
        broadcastState(gameRef.current);
      }
    });

    channel.on("presence", { event: "sync" }, () => {
      schedulePresenceReconciliation();
    });

    channel.on("presence", { event: "join" }, () => {
      schedulePresenceReconciliation();
    });

    channel.on("presence", { event: "leave" }, () => {
      schedulePresenceReconciliation();
    });

    channel.subscribe(async (status) => {
      if (status === "SUBSCRIBED") {
        await channel.track({ userId: myId });
        if (gameRef.current) {
          refreshPrivateChannels(gameRef.current.code, gameRef.current.players);
          if (!isHostRef.current) broadcastSyncRequest();
          else broadcastState(gameRef.current);
        }
        onConnected();
        schedulePresenceReconciliation();
      } else if (status === "CHANNEL_ERROR" || status === "TIMED_OUT" || status === "CLOSED") {
        if (!gameRef.current || reconnectTimerRef.current) return;
        reconnectTimerRef.current = setTimeout(() => {
          reconnectTimerRef.current = null;
          const current = gameRef.current;
          if (current) setupChannel(current.code, () => {
            if (!isHostRef.current) broadcastSyncRequest();
            else if (gameRef.current) broadcastState(gameRef.current);
          });
        }, 1500);
      }
    });

    channelRef.current = channel;
  }

  useEffect(() => {
    const sub = AppState.addEventListener("change", (nextState) => {
      if (nextState !== "active" || !gameRef.current || !supabase) return;
      const current = gameRef.current;
      setupChannel(current.code, () => {
        if (!isHostRef.current) broadcastSyncRequest();
        else if (gameRef.current) broadcastState(gameRef.current);
      });
    });
    return () => sub.remove();
  }, []);

  function beginGuest() {
    // Le mode sans compte doit fonctionner immédiatement.
    // Le prénom/pseudo sera demandé au moment de créer ou rejoindre une partie.
    setError("");
    if (!name.trim()) setName("Joueur");
    setScreen("home");
  }

  function createGame() {
    if (!name.trim()) {
      setError(t.name);
      return;
    }

    const host: Player = {
      id: myId,
      name: name.trim(),
      avatar,
    };

    const gameCode = generateCode();
    const initial = makeInitialState(gameCode, host);

    isHostRef.current = true;
    setLocalState(initial);

    setupChannel(gameCode, () => {
      if (gameRef.current) broadcastState(gameRef.current);
    });

    setScreen("game");
  }

  function joinGame() {
    if (!name.trim() || code.trim().length !== 4) {
      setError("Nom et code de partie requis.");
      return;
    }

    isHostRef.current = false;
    gameRef.current = null;
    setGame(null);

    const gameCode = code.trim().toUpperCase();

    const sendJoin = () => {
      channelRef.current?.send({
        type: "broadcast",
        event: "GAME_ACTION",
        payload: {
          type: "JOIN_REQUEST",
          player: {
            id: myId,
            name: name.trim(),
            avatar,
          },
        } satisfies Action,
      });
    };

    setupChannel(gameCode, sendJoin);
    setScreen("game");

    let attempts = 0;
    const timer = setInterval(() => {
      const current = gameRef.current;
      const joined = !!current?.players.some((p) => p.id === myId);
      const pending = !!current?.joinRequests.some((p) => p.id === myId);

      // Recevoir GAME_STATE ne signifie pas encore que la demande a été acceptée.
      // On arrête seulement lorsque le joueur est réellement membre de la salle.
      if (joined) {
        clearInterval(timer);
        return;
      }

      attempts += 1;
      if (!pending || attempts % 2 === 0) sendJoin();

      if (attempts >= 20) {
        clearInterval(timer);
        closeChannel();
        setGame(null);
        gameRef.current = null;
        setError("Partie introuvable ou demande non acceptée.");
        setScreen("join");
      }
    }, 1000);
  }

  function requestLeaveGame() {
    if (!game) return;
    Alert.alert(t.leaveConfirm, t.leaveConfirmText, [
      { text: t.stay, style: "cancel" },
      {
        text: t.leaveGame,
        style: "destructive",
        onPress: () => {
          const current = gameRef.current;
          if (!current) return;

          const activeTurn =
            current.phase === "preparation" ||
            current.phase === "play" ||
            current.phase === "reveal";
          const isActiveParticipant =
            activeTurn &&
            (current.currentPlayerId === myId || current.targetPlayerId === myId);

          // Règle du jeu : la cible / joueur actif ne peut pas quitter pendant son tour.
          if (isActiveParticipant) {
            Alert.alert("AVOU", "Tu ne peux pas quitter pendant ton tour. Termine le tour puis quitte la partie.");
            return;
          }

          const leavingHost = current.hostId === myId;
          const previousVersion = current.stateVersion;
          dispatch({ type: "LEAVE", playerId: myId });
          setShowInGameSettings(false);

          setTimeout(() => {
            const after = gameRef.current;
            const removed = !after?.players.some((p) => p.id === myId);

            if (removed) {
              closeChannel();
              gameRef.current = null;
              setGame(null);
              isHostRef.current = false;
              setScreen("home");
            } else if (leavingHost && after?.stateVersion === previousVersion) {
              // Le transfert n'a pas été accepté : on reste dans la partie plutôt que
              // de quitter localement et de laisser les autres dans un état incohérent.
              Alert.alert("AVOU", "Le départ n'a pas pu être synchronisé. Tu restes dans la partie.");
            }
          }, 1100);
        },
      },
    ]);
  }

  function leaveGame() {
    closeChannel();
    gameRef.current = null;
    setGame(null);
    isHostRef.current = false;
    setScreen("home");
  }

  function renderWelcome() {
    return (
      <View style={styles.center}>
        <View style={styles.logo}>
          <Text style={styles.logoText}>AVOU</Text>
        </View>

        <Text style={styles.heroTitle}>{t.subtitle}</Text>
        <Text style={styles.heroSub}>{t.accountOptional}</Text>

        <Button label={t.guest} onPress={beginGuest} variant="pink" />
        <TouchableOpacity
          style={styles.language}
          onPress={() => setScreen("language")}
        >
          <Text style={styles.languageText}>
            🌍 {LANGUAGES.find((x) => x.code === lang)?.label}
          </Text>
        </TouchableOpacity>

        {!!error && <Text style={styles.error}>{error}</Text>}
      </View>
    );
  }

  function renderHome() {
    return (
      <ScrollView contentContainerStyle={styles.scroll}>
        <View style={styles.header}>
          <View>
            <Text style={styles.brand}>AVOU</Text>
            <Text style={styles.headerSub}>{t.subtitle}</Text>
          </View>
          <TouchableOpacity
            style={styles.avatarSmall}
            onPress={() => setScreen("space")}
          >
            <Text>{avatar}</Text>
          </TouchableOpacity>
        </View>

        <View style={[styles.redHero, { backgroundColor: liveTheme.hero }]}>
          <Text style={styles.redHeroTitle}>ACTION.</Text>
          <Text style={styles.redHeroTitle}>VÉRITÉ.</Text>
          <Text style={styles.redHeroSub}>Et personne ne sait ce qui arrive.</Text>
        </View>

        <View style={styles.modesInfoCard}>
          <Text style={styles.sectionTitle}>🎮 MODES DE JEU</Text>
          <Text style={styles.modeInfoTitle}>🎯 MODE CIBLE — 2 joueurs minimum</Text>
          <Text style={styles.modeInfoText}>{t.targetModeHelp}</Text>
          <Text style={styles.modeInfoTitle}>🔥 MODE INTENSE — 3 joueurs minimum</Text>
          <Text style={styles.modeInfoText}>{t.intenseModeHelp}</Text>
        </View>

        <Button label={t.create} onPress={() => setScreen("create")} variant="pink" />
        <Button label={t.join} onPress={() => setScreen("join")} />
        <Button label={t.space} onPress={() => setScreen("space")} variant="yellow" />
        <Button label={`🌍 ${t.language}`} onPress={() => setScreen("language")} variant="outline" />

        {!!error && <Text style={styles.error}>{error}</Text>}
      </ScrollView>
    );
  }

  function renderCreate() {
    return (
      <ScrollView contentContainerStyle={styles.scroll}>
        <Text style={styles.screenTitle}>{t.create}</Text>

        <Text style={styles.label}>{t.name}</Text>
        <TextInput
          value={name}
          onChangeText={setName}
          placeholder={t.name}
          placeholderTextColor="#9D91A9"
          style={styles.input}
        />

        <Text style={styles.label}>Avatar</Text>
        <View style={styles.avatarGrid}>
          {AVATARS.map((item) => (
            <TouchableOpacity
              key={item}
              onPress={() => setAvatar(item)}
              style={[
                styles.avatarChoice,
                avatar === item && styles.avatarChoiceSelected,
              ]}
            >
              <Text style={styles.avatarEmoji}>{item}</Text>
            </TouchableOpacity>
          ))}
        </View>

        <View style={styles.modesInfoCard}>
          <Text style={styles.sectionTitle}>🎮 CHOISISSEZ LE MODE</Text>
          <Text style={styles.modeInfoTitle}>🎯 MODE CIBLE — 2 joueurs minimum</Text>
          <Text style={styles.modeInfoText}>{t.targetModeHelp}</Text>
          <Text style={styles.modeInfoTitle}>🔥 MODE INTENSE — 3 joueurs minimum</Text>
          <Text style={styles.modeInfoText}>{t.intenseModeHelp}</Text>
          <Text style={styles.helper}>Le Maître de jeu choisit le mode. À 2 joueurs, seul le Mode Cible est disponible.</Text>
        </View>

        <Button label={t.createNow} onPress={createGame} variant="pink" />
        <Button label={t.back} onPress={() => setScreen("home")} variant="outline" />
        {!!error && <Text style={styles.error}>{error}</Text>}
      </ScrollView>
    );
  }

  function renderJoin() {
    return (
      <ScrollView contentContainerStyle={styles.scroll}>
        <Text style={styles.screenTitle}>{t.join}</Text>

        <Text style={styles.label}>{t.name}</Text>
        <TextInput
          value={name}
          onChangeText={setName}
          placeholder={t.name}
          placeholderTextColor="#9D91A9"
          style={styles.input}
        />

        <Text style={styles.label}>{t.code}</Text>
        <TextInput
          value={code}
          onChangeText={(v) => setCode(v.toUpperCase())}
          maxLength={4}
          autoCapitalize="characters"
          placeholder="ABCD"
          placeholderTextColor="#9D91A9"
          style={[styles.input, styles.codeInput]}
        />

        <Button label={t.joinNow} onPress={joinGame} variant="pink" />
        <Button label={t.back} onPress={() => setScreen("home")} variant="outline" />
        {!!error && <Text style={styles.error}>{error}</Text>}
      </ScrollView>
    );
  }

  function renderLanguage() {
    return (
      <ScrollView contentContainerStyle={styles.scroll}>
        <Text style={styles.screenTitle}>{t.language}</Text>

        {LANGUAGES.map((item) => (
          <TouchableOpacity
            key={item.code}
            onPress={() => {
              setLang(item.code);
              setScreen("home");
            }}
            style={[
              styles.languageRow,
              lang === item.code && styles.languageRowSelected,
            ]}
          >
            <Text style={styles.languageRowText}>{item.label}</Text>
          </TouchableOpacity>
        ))}

        <Button label={t.back} onPress={() => setScreen("home")} variant="outline" />
      </ScrollView>
    );
  }

  function renderAvatar() {
    return (
      <View style={styles.center}>
        <Text style={styles.screenTitle}>{t.avatar}</Text>

        <View style={styles.avatarBig}>
          <Text style={styles.avatarBigText}>{avatarChoice}</Text>
        </View>

        <View style={styles.avatarGrid}>
          {AVATARS.map((item) => (
            <TouchableOpacity
              key={item}
              onPress={() => setAvatarChoice(item)}
              style={[
                styles.avatarChoice,
                avatarChoice === item && styles.avatarChoiceSelected,
              ]}
            >
              <Text style={styles.avatarEmoji}>{item}</Text>
            </TouchableOpacity>
          ))}
        </View>

        <Button
          label={t.saveAvatar}
          onPress={() => {
            setAvatar(avatarChoice);
            setScreen("home");
          }}
          variant="pink"
        />
        <Button
          label={t.skip}
          onPress={() => setScreen("home")}
          variant="outline"
        />
      </View>
    );
  }

  function renderSpace() {
    return (
      <ScrollView contentContainerStyle={styles.scroll}>
        <Text style={styles.screenTitle}>📖 {t.space}</Text>

        <View style={styles.goodCard}>
          <Text style={styles.goodEmoji}>🎯</Text>
          <Text style={styles.goodTitle}>LES RÈGLES D’AVOU</Text>
          <Text style={styles.goodText}>
            Tout est expliqué ici pour que chaque partie soit simple, claire et sans ambiguïté.
          </Text>
        </View>

        <Text style={styles.sectionTitle}>🎯 MODE CIBLE</Text>
        <View style={styles.ruleCard}>
          <Text style={styles.ruleTitle}>À partir de 2 joueurs</Text>
          <Text style={styles.ruleText}>
            Le joueur dont c’est le tour choisit un autre joueur comme cible.
            Il lui propose ensuite une seule chose : une ACTION ou une VÉRITÉ.
          </Text>
          <Text style={styles.ruleText}>
            La cible voit uniquement le type de proposition avant de choisir.
            Elle choisit, puis le contenu est révélé. La cible réalise l’Action ou répond à la Vérité.
          </Text>
          <Text style={styles.ruleText}>
            Il n’y a pas de choix « Jouer ou Cibler » : en Mode Cible, le joueur actif cible directement un autre joueur.
          </Text>
        </View>

        <Text style={styles.sectionTitle}>🔥 MODE INTENSE</Text>
        <View style={styles.ruleCard}>
          <Text style={styles.ruleTitle}>À partir de 3 joueurs</Text>
          <Text style={styles.ruleText}>
            Tous les autres joueurs proposent une ACTION et une VÉRITÉ pour la cible.
          </Text>
          <Text style={styles.ruleText}>
            La cible voit uniquement les dos des propositions et leur type ACTION ou VÉRITÉ.
            Elle ne voit pas leur contenu avant son choix.
          </Text>
          <Text style={styles.ruleText}>
            La cible choisit une seule proposition. Son contenu est alors révélé, puis elle réalise l’Action ou répond à la Vérité.
          </Text>
          <Text style={styles.ruleText}>
            Toutes les propositions du tour sont ensuite supprimées. Il faut de nouvelles propositions au tour suivant.
          </Text>
        </View>

        <Text style={styles.sectionTitle}>🕵️ TENTATIVE DE DÉMASQUAGE — MODE INTENSE</Text>
        <View style={styles.ruleCard}>
          <Text style={styles.ruleTitle}>Option activable par le Maître de jeu</Text>
          <Text style={styles.ruleText}>
            Après chaque tour Intense, une tentative de démasquage peut être activée. Tous les joueurs votent pour deviner qui a écrit la proposition choisie. La cible vote aussi.
          </Text>
          <Text style={styles.ruleText}>
            Pendant le vote, les auteurs restent cachés. Une fois que tout le monde a voté, le résultat est révélé, puis le tour suivant commence.
          </Text>
        </View>

        <Text style={styles.sectionTitle}>👥 NOMBRE DE JOUEURS</Text>
        <View style={styles.ruleCard}>
          <Text style={styles.ruleTitle}>2 joueurs</Text>
          <Text style={styles.ruleText}>Le Mode Cible est disponible. Le Mode Intense n’est pas disponible.</Text>
          <Text style={styles.ruleTitle}>3 joueurs ou plus</Text>
          <Text style={styles.ruleText}>Les deux modes sont disponibles. Le Maître de jeu choisit le mode.</Text>
          <Text style={styles.ruleTitle}>De 3 à 2 joueurs</Text>
          <Text style={styles.ruleText}>Si la partie est en Mode Intense, elle passe automatiquement en Mode Cible.</Text>
          <Text style={styles.ruleTitle}>De 2 à 3 joueurs</Text>
          <Text style={styles.ruleText}>Le Mode Intense devient disponible, mais la partie ne bascule pas automatiquement dessus.</Text>
        </View>

        <Text style={styles.sectionTitle}>👁️ IDENTITÉS</Text>
        <View style={styles.ruleCard}>
          <Text style={styles.ruleTitle}>Mode Cible : toujours révélées</Text>
          <Text style={styles.ruleText}>
            En Mode Cible, les identités sont toujours révélées. Aucun réglage pour les cacher n’apparaît dans ce mode.
          </Text>
          <Text style={styles.ruleTitle}>Mode Intense</Text>
          <Text style={styles.ruleText}>
            Le Maître de jeu peut choisir : identités révélées ou identités cachées.
          </Text>
          <Text style={styles.ruleTitle}>Aucun vote</Text>
          <Text style={styles.ruleText}>
            Les réglages d’identité sont décidés par le Maître de jeu, pas par un vote des joueurs. En Mode Intense, le Maître de jeu peut aussi activer la « Tentative de démasquage ».
          </Text>
        </View>

        <Text style={styles.sectionTitle}>👑 MAÎTRE DE JEU</Text>
        <View style={styles.ruleCard}>
          <Text style={styles.ruleText}>
            Le créateur de la partie est le Maître de jeu. Il contrôle les réglages de la partie et choisit le mode lorsqu’il y a plusieurs possibilités.
          </Text>
          <Text style={styles.ruleText}>
            Il peut également gérer les joueurs, accepter ou refuser une demande de rejoindre et modifier les réglages prévus pour le Maître de jeu.
          </Text>
          <Text style={styles.ruleText}>
            Si le Maître de jeu quitte la partie, un nouveau Maître de jeu est automatiquement désigné parmi les joueurs restants. La partie, le code et le tour en cours continuent.
          </Text>
        </View>

        <Text style={styles.sectionTitle}>🔢 ORDRE / DÉSORDRE</Text>
        <View style={styles.ruleCard}>
          <Text style={styles.ruleTitle}>ORDRE</Text>
          <Text style={styles.ruleText}>Les joueurs passent dans un ordre fixe.</Text>
          <Text style={styles.ruleTitle}>DÉSORDRE</Text>
          <Text style={styles.ruleText}>L’application choisit aléatoirement le prochain joueur, en évitant autant que possible de reprendre immédiatement le même joueur.</Text>
          <Text style={styles.ruleText}>Le choix ORDRE / DÉSORDRE appartient au Maître de jeu.</Text>
        </View>

        <Text style={styles.sectionTitle}>🔒 CE QUI RESTE CACHÉ</Text>
        <View style={styles.ruleCard}>
          <Text style={styles.ruleText}>
            Dans le Mode Intense, la cible ne voit jamais le contenu des propositions avant d’avoir choisi. Elle voit uniquement leur type.
          </Text>
          <Text style={styles.ruleText}>
            Les propositions non choisies ne sont pas conservées pour plus tard. Elles sont supprimées à la fin du tour.
          </Text>
        </View>

        <Text style={styles.sectionTitle}>🚪 REJOINDRE OU QUITTER</Text>
        <View style={styles.ruleCard}>
          <Text style={styles.ruleText}>
            Un joueur peut rejoindre une partie avec le code de la salle, après acceptation du Maître de jeu. Une arrivée en cours de partie ne redémarre pas la partie.
          </Text>
          <Text style={styles.ruleText}>
            Un joueur peut quitter la partie. Si le nombre de joueurs reste suffisant, la partie continue.
          </Text>
          <Text style={styles.ruleText}>
            Le joueur actif ou la cible ne peut pas quitter pendant un tour actif.
          </Text>
          <Text style={styles.ruleText}>
            À un seul joueur restant, la partie s’arrête.
          </Text>
        </View>

        <Text style={styles.sectionTitle}>💬 MESSAGES PRIVÉS</Text>
        <View style={styles.ruleCard}>
          <Text style={styles.ruleText}>
            Chaque joueur peut envoyer un message privé à un autre joueur à tout moment dans la salle. Seuls l’expéditeur et le destinataire voient la conversation.
          </Text>
        </View>

        <View style={styles.goodCard}>
          <Text style={styles.goodEmoji}>✨</Text>
          <Text style={styles.goodTitle}>UNE RÈGLE SIMPLE</Text>
          <Text style={styles.goodText}>
            Chaque tour recommence avec de nouvelles propositions. Rien n’est gardé pour un tour suivant.
          </Text>
        </View>

        <Button label={t.back} onPress={() => setScreen("home")} variant="outline" />
      </ScrollView>
    );
  }

  function renderLobby() {
    if (!game) return null;
    const isHost = game.hostId === myId;

    return (
      <ScrollView contentContainerStyle={styles.scroll}>
        <Text style={styles.screenTitle}>{t.lobby}</Text>

        <View style={styles.codeCard}>
          <Text style={styles.codeLabel}>{t.code}</Text>
          <Text style={styles.bigCode}>{game.code}</Text>
        </View>

        <Text style={styles.sectionTitle}>
          {game.players.length} joueur{game.players.length > 1 ? "s" : ""}
        </Text>

        {game.players.map((p) => (
          <View key={p.id} style={styles.playerRow}>
            <Text style={styles.playerAvatar}>{p.avatar}</Text>
            <View style={styles.playerInfo}>
              <Text style={styles.playerName}>{p.name}</Text>
              {p.id === game.hostId && (
                <Text style={styles.hostBadge}>👑 {t.host}</Text>
              )}
            </View>
            {isHost && p.id !== game.hostId && (
              <TouchableOpacity onPress={() => dispatch({ type: "REMOVE_PLAYER", playerId: myId, targetPlayerId: p.id })}>
                <Text style={styles.removeText}>✕ {t.removePlayer}</Text>
              </TouchableOpacity>
            )}
          </View>
        ))}

        {isHost && (
          <View style={styles.requestsBox}>
            <Text style={styles.sectionTitle}>{t.gameMode}</Text>
            <TouchableOpacity
              onPress={() => dispatch({ type: "SET_GAME_MODE", playerId: myId, mode: "target" })}
              style={[styles.ruleCard, game.gameMode === "target" && styles.ruleCardSelected]}
            >
              <Text style={styles.ruleTitle}>{t.targetMode}</Text>
              <Text style={styles.ruleText}>{t.targetModeHelp}</Text>
              <Text style={styles.helper}>👥 2 joueurs minimum</Text>
            </TouchableOpacity>
            <TouchableOpacity
              disabled={game.players.length < 3}
              onPress={() => dispatch({ type: "SET_GAME_MODE", playerId: myId, mode: "intense" })}
              style={[styles.ruleCard, game.gameMode === "intense" && styles.ruleCardSelected, game.players.length < 3 && styles.ruleCardDisabled]}
            >
              <Text style={styles.ruleTitle}>{t.intenseMode}</Text>
              <Text style={styles.ruleText}>{t.intenseModeHelp}</Text>
              <Text style={styles.helper}>👥 3 joueurs minimum</Text>
            </TouchableOpacity>
          </View>
        )}

        {isHost && game.joinRequests.length > 0 && (
          <View style={styles.requestsBox}>
            <Text style={styles.sectionTitle}>👑 {t.pendingJoin}</Text>
            {game.joinRequests.map((request) => (
              <View key={request.id} style={styles.requestRow}>
                <Text style={styles.playerName}>{request.avatar} {request.name}</Text>
                <View style={styles.requestButtons}>
                  <Button label={t.accept} onPress={() => dispatch({ type: "APPROVE_JOIN", playerId: myId, requesterId: request.id })} variant="pink" />
                  <Button label={t.reject} onPress={() => dispatch({ type: "REJECT_JOIN", playerId: myId, requesterId: request.id })} variant="outline" />
                </View>
              </View>
            ))}
          </View>
        )}

        {isHost ? (
          <Button
            label={game.players.length >= minimumPlayersForMode(game.gameMode) ? t.start : t.need3}
            onPress={() => dispatch({ type: "START_SETUP", playerId: myId })}
            disabled={game.players.length < minimumPlayersForMode(game.gameMode)}
            variant="pink"
          />
        ) : (
          <View style={styles.waitCard}>
            <Text style={styles.waitText}>{t.waiting}</Text>
          </View>
        )}

        <Button label={t.back} onPress={leaveGame} variant="outline" />
      </ScrollView>
    );
  }

  function renderSettings() {
    if (!game) return null;
    const isHost = game.hostId === myId;

    return (
      <ScrollView contentContainerStyle={styles.scroll}>
        <Text style={styles.screenTitle}>{t.settings}</Text>

        {game.gameMode === "intense" && (
          <>
            <Text style={styles.sectionTitle}>{t.identities}</Text>
            <Text style={styles.helper}>
              Seul le Maître de jeu décide. Il n'y a aucun vote.
            </Text>

            {(
              [
                ["reveal", t.revealed],
                ["hidden", t.hidden],
              ] as [IdentityMode, string][]
            ).map(([mode, label]) => (
              <TouchableOpacity
                key={mode}
                disabled={!isHost}
                onPress={() =>
                  dispatch({
                    type: "SET_IDENTITY_MODE",
                    playerId: myId,
                    mode,
                  })
                }
                style={[
                  styles.ruleCard,
                  game.identityMode === mode && styles.ruleCardSelected,
                ]}
              >
                <Text style={styles.ruleTitle}>{label}</Text>
                <Text style={styles.ruleText}>
                  {mode === "reveal"
                    ? "Les identités sont visibles pendant les tours."
                    : "Les identités restent cachées pendant le tour."}
                </Text>
              </TouchableOpacity>
            ))}
          </>
        )}

        <Text style={styles.sectionTitle}>{t.gameMode}</Text>
        <Text style={styles.helper}>Le Maître de jeu choisit le mode pour toute la partie.</Text>

        <TouchableOpacity
          disabled={!isHost}
          onPress={() =>
            dispatch({ type: "SET_GAME_MODE", playerId: myId, mode: "target" })
          }
          style={[
            styles.ruleCard,
            game.gameMode === "target" && styles.ruleCardSelected,
          ]}
        >
          <Text style={styles.ruleTitle}>{t.targetMode}</Text>
          <Text style={styles.ruleText}>{t.targetModeHelp}</Text>
              <Text style={styles.helper}>👥 2 joueurs minimum</Text>
        </TouchableOpacity>

        <TouchableOpacity
          disabled={!isHost}
          onPress={() =>
            dispatch({ type: "SET_GAME_MODE", playerId: myId, mode: "intense" })
          }
          style={[
            styles.ruleCard,
            game.gameMode === "intense" && styles.ruleCardSelected,
          ]}
        >
          <Text style={styles.ruleTitle}>{t.intenseMode}</Text>
          <Text style={styles.ruleText}>{t.intenseModeHelp}</Text>
              <Text style={styles.helper}>👥 3 joueurs minimum</Text>
        </TouchableOpacity>

        {game.gameMode === "intense" && isHost && (
          <TouchableOpacity
            onPress={() => dispatch({ type: "SET_DEMASKING_ENABLED", playerId: myId, enabled: !game.demaskingEnabled })}
            style={[styles.ruleCard, game.demaskingEnabled && styles.ruleCardSelected]}
          >
            <Text style={styles.ruleTitle}>{game.demaskingEnabled ? "🕵️ TENTATIVE DE DÉMASQUAGE : ACTIVÉE" : "🕵️ TENTATIVE DE DÉMASQUAGE : DÉSACTIVÉE"}</Text>
            <Text style={styles.ruleText}>Après chaque tour Intense, tous les joueurs votent pour deviner qui a écrit la proposition choisie. La cible vote aussi.</Text>
          </TouchableOpacity>
        )}

        {isHost && (
          <Button
            label="Continuer → Ordre / Désordre"
            onPress={() =>
              dispatch({
                type: "START_ORDER_CHOICE",
                playerId: myId,
              })
            }
            variant="pink"
          />
        )}

        {!isHost && (
          <View style={styles.waitCard}>
            <Text style={styles.waitText}>{t.waiting}</Text>
          </View>
        )}
      </ScrollView>
    );
  }

  function renderOrderChoice() {
    if (!game) return null;
    const isHost = game.hostId === myId;

    return (
      <ScrollView contentContainerStyle={styles.scroll}>
        <Text style={styles.screenTitle}>ORDRE / DÉSORDRE</Text>

        <TouchableOpacity
          disabled={!isHost}
          onPress={() =>
            dispatch({
              type: "SET_ORDER",
              playerId: myId,
              mode: "order",
            })
          }
          style={[
            styles.ruleCard,
            game.orderMode === "order" && styles.ruleCardSelected,
          ]}
        >
          <Text style={styles.ruleTitle}>🔢 {t.order}</Text>
          <Text style={styles.ruleText}>{t.orderHelp}</Text>
        </TouchableOpacity>

        <TouchableOpacity
          disabled={!isHost}
          onPress={() =>
            dispatch({
              type: "SET_ORDER",
              playerId: myId,
              mode: "disorder",
            })
          }
          style={[
            styles.ruleCard,
            game.orderMode === "disorder" && styles.ruleCardSelected,
          ]}
        >
          <Text style={styles.ruleTitle}>🎲 {t.disorder}</Text>
          <Text style={styles.ruleText}>{t.disorderHelp}</Text>
        </TouchableOpacity>

        {!isHost && (
          <View style={styles.waitCard}>
            <Text style={styles.waitText}>{t.waiting}</Text>
          </View>
        )}
      </ScrollView>
    );
  }

  function renderIntenseHeader(title: string, subtitle: string, eyebrow = "MODE INTENSE") {
    return (
      <Animated.View
        style={[
          styles.intenseHero,
          {
            transform: [{ scale: intensePulse.interpolate({ inputRange: [0, 1], outputRange: [1, 1.012] }) }],
          },
        ]}
      >
        <View style={styles.intenseHeroGlow} />
        <View style={styles.intenseHeroTopRow}>
          <View style={styles.intensePill}><Text style={styles.intensePillText}>🔥 {eyebrow}</Text></View>
          <Text style={styles.intenseHeroSpark}>✦</Text>
        </View>
        <Text style={styles.intenseHeroTitle}>{title}</Text>
        <Text style={styles.intenseHeroSub}>{subtitle}</Text>
      </Animated.View>
    );
  }

  function renderIntenseCardBack(card: Proposal, index: number) {
    return (
      <TouchableOpacity
        activeOpacity={0.88}
        onPress={() => dispatch({ type: "SELECT_PROPOSAL", playerId: myId, proposalId: card.id })}
        style={[styles.intenseCard, index % 2 === 0 ? styles.intenseCardAction : styles.intenseCardTruth]}
      >
        <View style={styles.intenseCardShine} />
        <Text style={styles.intenseCardMini}>AVOU • PROPOSITION</Text>
        <View style={styles.intenseCardCenter}>
          <Text style={styles.intenseCardIcon}>{card.type === "action" ? "A" : "V"}</Text>
          <Text style={styles.intenseCardType}>{card.type === "action" ? "ACTION" : "VÉRITÉ"}</Text>
          <Text style={styles.intenseCardHint}>Le contenu reste secret</Text>
        </View>
        <Text style={styles.intenseCardNumber}>{String(index + 1).padStart(2, "0")}</Text>
      </TouchableOpacity>
    );
  }

  function renderPreparation() {
    if (!game) return null;

    if (game.gameMode === "target") {
      const amCurrent = game.currentPlayerId === myId;
      const alreadySubmitted = game.submittedForTurn.includes(myId);
      const others = game.players.filter((p) => p.id !== myId);

      if (!amCurrent) {
        return (
          <ScrollView contentContainerStyle={styles.scroll}>
            <View style={[styles.redHero, { backgroundColor: liveTheme.hero }]}>
              <Text style={styles.redHeroTitle}>🎯 MODE CIBLE</Text>
              <Text style={styles.redHeroSub}>
                {currentPlayer?.name ?? "Un joueur"} prépare sa cible.
              </Text>
            </View>
            <View style={styles.waitCard}>
              <Text style={styles.waitText}>{t.waiting}</Text>
            </View>
            </ScrollView>
        );
      }

      if (alreadySubmitted) {
        return (
          <ScrollView contentContainerStyle={styles.scroll}>
            <View style={styles.goodCard}>
              <Text style={styles.goodEmoji}>🎯</Text>
              <Text style={styles.goodTitle}>Cible choisie</Text>
              <Text style={styles.goodText}>La cible va recevoir ta carte.</Text>
            </View>
            <View style={styles.waitCard}>
              <Text style={styles.waitText}>{t.waiting}</Text>
            </View>
          </ScrollView>
        );
      }

      return (
        <ScrollView contentContainerStyle={styles.scroll}>
          <View style={[styles.redHero, { backgroundColor: liveTheme.hero }]}>
            <Text style={styles.redHeroTitle}>🎯 MODE CIBLE</Text>
            <Text style={styles.redHeroSub}>
              Choisis directement la personne qui devra jouer.
            </Text>
          </View>

          <Text style={styles.sectionTitle}>{t.chooseTarget}</Text>
          <View style={styles.targetGrid}>
            {others.map((p) => (
              <TouchableOpacity
                key={p.id}
                onPress={() => setTargetChoice(p.id)}
                style={[
                  styles.targetChoice,
                  targetChoice === p.id && styles.targetChoiceSelected,
                ]}
              >
                <Text style={styles.targetChoiceAvatar}>{p.avatar}</Text>
                <Text style={styles.targetChoiceName}>{p.name}</Text>
              </TouchableOpacity>
            ))}
          </View>

          <Text style={styles.sectionTitle}>{t.proposalType}</Text>
          <View style={styles.typeRow}>
            <TouchableOpacity
              onPress={() => setTargetType("action")}
              style={[
                styles.typeChoice,
                targetType === "action" && styles.typeChoiceSelected,
              ]}
            >
              <Text style={styles.typeChoiceEmoji}>🔥</Text>
              <Text style={styles.typeChoiceText}>ACTION</Text>
            </TouchableOpacity>
            <TouchableOpacity
              onPress={() => setTargetType("truth")}
              style={[
                styles.typeChoice,
                targetType === "truth" && styles.typeChoiceSelected,
              ]}
            >
              <Text style={styles.typeChoiceEmoji}>💬</Text>
              <Text style={styles.typeChoiceText}>VÉRITÉ</Text>
            </TouchableOpacity>
          </View>

          <Text style={styles.label}>{t.writeProposal}</Text>
          <TextInput
            value={targetProposalText}
            onChangeText={setTargetProposalText}
            multiline
            placeholder={targetType === "action" ? "Écris une Action…" : "Écris une Vérité…"}
            placeholderTextColor="#9D91A9"
            style={[styles.input, styles.textArea]}
          />

          <Button
            label={t.confirmTarget}
            disabled={!targetChoice || !targetProposalText.trim()}
            onPress={() => {
              if (!targetChoice || !targetProposalText.trim()) return;
              dispatch({
                type: "SUBMIT_TARGET_PROPOSAL",
                playerId: myId,
                targetPlayerId: targetChoice,
                typeChoice: targetType,
                text: targetProposalText,
              });
              setTargetProposalText("");
              setTargetChoice(null);
              setTargetType("truth");
            }}
            variant="pink"
          />
        </ScrollView>
      );
    }

    const target = currentPlayer;
    const amTarget = game.currentPlayerId === myId;
    const alreadySubmitted = game.submittedForTurn.includes(myId);

    if (game.gameMode === "intense") {
      if (amTarget) {
        return (
          <ScrollView contentContainerStyle={styles.intenseScroll}>
            {renderIntenseHeader("LA TENSION MONTE", "Cette manche est pour toi. Les autres joueurs préparent leurs propositions.")}
            <View style={styles.intenseTargetFocus}>
              <Text style={styles.intenseTargetKicker}>CIBLE DE LA MANCHE</Text>
              <Text style={styles.intenseTargetAvatar}>{target?.avatar ?? "✦"}</Text>
              <Text style={styles.intenseTargetName}>{target?.name ?? "Toi"}</Text>
              <Text style={styles.intenseTargetHint}>Tu découvriras les propositions uniquement après leur validation.</Text>
            </View>
            <View style={styles.intenseWaitPanel}>
              <Text style={styles.intenseWaitTitle}>Les cartes se préparent…</Text>
              <Text style={styles.intenseWaitText}>Chaque joueur écrit une Action et une Vérité. Rien ne sera conservé après cette manche.</Text>
            </View>
          </ScrollView>
        );
      }

      if (alreadySubmitted) {
        return (
          <ScrollView contentContainerStyle={styles.intenseScroll}>
            {renderIntenseHeader("C'EST ENVOYÉ", "Ta contribution est prête. Maintenant, laisse le suspense faire son travail.")}
            <View style={styles.intenseSubmittedPanel}>
              <Text style={styles.intenseSubmittedIcon}>✓</Text>
              <Text style={styles.intenseSubmittedTitle}>Action + Vérité envoyées</Text>
              <Text style={styles.intenseSubmittedText}>Elles seront mélangées avec celles des autres joueurs.</Text>
            </View>
          </ScrollView>
        );
      }

      return (
        <ScrollView contentContainerStyle={styles.intenseScroll}>
          {renderIntenseHeader("À TOI DE JOUER", `Propose une Action et une Vérité à ${target?.name ?? "la cible"}.`, "HOT VIBE")}
          <View style={styles.intenseTargetMini}>
            <Text style={styles.intenseTargetMiniAvatar}>{target?.avatar ?? "✦"}</Text>
            <View style={{ flex: 1 }}>
              <Text style={styles.intenseTargetMiniLabel}>POUR LA CIBLE</Text>
              <Text style={styles.intenseTargetMiniName}>{target?.name ?? "Joueur"}</Text>
            </View>
            <Text style={styles.intenseTargetMiniSpark}>✦</Text>
          </View>

          <View style={styles.intenseInputBlock}>
            <View style={styles.intenseInputHeading}>
              <View style={[styles.intenseTypeBadge, styles.intenseActionBadge]}><Text style={styles.intenseTypeBadgeText}>A</Text></View>
              <View style={{ flex: 1 }}><Text style={styles.intenseInputTitle}>ACTION</Text><Text style={styles.intenseInputSub}>Donne envie d'oser.</Text></View>
            </View>
            <TextInput value={actionText} onChangeText={setActionText} multiline placeholder="Écris une Action…" placeholderTextColor="#A36B76" style={styles.intenseTextArea} />
          </View>

          <View style={styles.intenseInputBlock}>
            <View style={styles.intenseInputHeading}>
              <View style={[styles.intenseTypeBadge, styles.intenseTruthBadge]}><Text style={styles.intenseTypeBadgeText}>V</Text></View>
              <View style={{ flex: 1 }}><Text style={styles.intenseInputTitle}>VÉRITÉ</Text><Text style={styles.intenseInputSub}>Pose la question qui intrigue.</Text></View>
            </View>
            <TextInput value={truthText} onChangeText={setTruthText} multiline placeholder="Écris une Vérité…" placeholderTextColor="#A36B76" style={styles.intenseTextArea} />
          </View>

          <TouchableOpacity
            disabled={!actionText.trim() || !truthText.trim()}
            activeOpacity={0.88}
            onPress={() => {
              dispatch({ type: "SUBMIT_PROPOSALS", playerId: myId, actionText, truthText });
              setActionText(""); setTruthText("");
            }}
            style={[styles.intenseMainButton, (!actionText.trim() || !truthText.trim()) && styles.intenseMainButtonDisabled]}
          >
            <Text style={styles.intenseMainButtonText}>ENVOYER MES PROPOSITIONS</Text>
            <Text style={styles.intenseMainButtonArrow}>→</Text>
          </TouchableOpacity>
        </ScrollView>
      );
    }

    if (amTarget) {
      return (
        <ScrollView contentContainerStyle={styles.scroll}>
          <View style={[styles.redHero, { backgroundColor: liveTheme.hero }]}>
            <Text style={styles.redHeroTitle}>{t.target}</Text>
            <Text style={styles.redHeroSub}>C'est ton tour.</Text>
          </View>
          <Text style={styles.sectionTitle}>{t.waiting}</Text>
          <View style={styles.waitCard}><Text style={styles.waitText}>Les autres joueurs préparent tes propositions.</Text></View>
        </ScrollView>
      );
    }

    if (alreadySubmitted) {
      return (
        <ScrollView contentContainerStyle={styles.scroll}>
          <Text style={styles.screenTitle}>{t.prepare}</Text>
          <View style={styles.waitCard}><Text style={styles.waitText}>{t.waiting}{"\n"}Tu as déjà envoyé ton Action et ta Vérité pour{target ? ` ${target.name}` : ""}.</Text></View>
          <Text style={styles.helper}>Rien ne sera conservé pour le prochain tour.</Text>
        </ScrollView>
      );
    }

    return (
      <ScrollView contentContainerStyle={styles.scroll}>
        <Text style={styles.screenTitle}>{t.prepare}</Text>
        <View style={styles.targetCard}><Text style={styles.targetEmoji}>{target?.avatar ?? "🎯"}</Text><Text style={styles.targetTitle}>{target?.name}</Text><Text style={styles.targetSub}>{t.target}</Text></View>
        <Text style={styles.helper}>{t.prepareHelp}</Text>
        <Text style={styles.label}>{t.action}</Text>
        <TextInput value={actionText} onChangeText={setActionText} multiline placeholder="Écris une Action…" placeholderTextColor="#9D91A9" style={[styles.input, styles.textArea]} />
        <Text style={styles.label}>{t.truth}</Text>
        <TextInput value={truthText} onChangeText={setTruthText} multiline placeholder="Écris une Vérité…" placeholderTextColor="#9D91A9" style={[styles.input, styles.textArea]} />
        <Button label={t.send} disabled={!actionText.trim() || !truthText.trim()} onPress={() => { dispatch({ type: "SUBMIT_PROPOSALS", playerId: myId, actionText, truthText }); setActionText(""); setTruthText(""); }} variant="pink" />
      </ScrollView>
    );
  }

  function renderPlay() {
    if (!game) return null;

    if (game.gameMode === "target") {
      const isTarget = game.targetPlayerId === myId;
      const target = game.players.find((p) => p.id === game.targetPlayerId);

      if (!isTarget) {
        return (
          <ScrollView contentContainerStyle={styles.scroll}>
            <View style={styles.waitCard}>
              <Text style={styles.turnEmoji}>🎯</Text>
              <Text style={styles.turnTitle}>{target?.name ?? "Cible"}</Text>
              <Text style={styles.waitText}>
                {currentPlayer?.name ?? "Le joueur"} a choisi cette cible.
              </Text>
            </View>
            </ScrollView>
        );
      }

      const card = game.proposals[0];
      return (
        <ScrollView contentContainerStyle={styles.scroll}>
          <View style={[styles.redHero, { backgroundColor: liveTheme.hero }]}>
            <Text style={styles.redHeroTitle}>🎯 À TOI</Text>
            <Text style={styles.redHeroSub}>Tu as été ciblé(e). Choisis ta carte.</Text>
          </View>
          <View style={[styles.cardGrid, { backgroundColor: "transparent" }]}>
            {card && (
              <CardBack
                type={card.type}
                index={0}
                onPress={() =>
                  dispatch({ type: "SELECT_PROPOSAL", playerId: myId, proposalId: card.id })
                }
                actionBackground={liveTheme.cardAction}
                actionBorder={liveTheme.primary}
                truthBackground={liveTheme.cardTruth}
                truthBorder={liveTheme.secondary}
              />
            )}
          </View>
        </ScrollView>
      );
    }

    if (game.gameMode === "intense") {
      if (!isMyTurn) {
        return (
          <ScrollView contentContainerStyle={styles.intenseScroll}>
            {renderIntenseHeader("LE TOUR DE", `${currentPlayer?.name ?? "un joueur"}`, "INTENSE")}
            <View style={styles.intenseSpectator}>
              <Text style={styles.intenseSpectatorIcon}>◉</Text>
              <Text style={styles.intenseSpectatorTitle}>Regarde bien.</Text>
              <Text style={styles.intenseSpectatorText}>Les propositions sont prêtes. La cible va bientôt choisir sans connaître leur contenu.</Text>
            </View>
            <View style={styles.intenseRoundPill}><Text style={styles.intenseRoundPillText}>MANCHE {game.round} / {game.maxRounds}</Text></View>
          </ScrollView>
        );
      }

      const cards = game.proposals;
      return (
        <ScrollView contentContainerStyle={styles.intenseScroll}>
          {renderIntenseHeader("CHOISIS TON FRISSON", "Tu connais le type. Pas le contenu. À toi de décider.", "À TOI DE CHOISIR")}
          <View style={styles.intenseChoiceMeta}>
            <View><Text style={styles.intenseMetaLabel}>MANCHE</Text><Text style={styles.intenseMetaValue}>{game.round} / {game.maxRounds}</Text></View>
            <View style={styles.intenseMetaDivider} />
            <View><Text style={styles.intenseMetaLabel}>IDENTITÉ</Text><Text style={styles.intenseMetaValue}>{game.roundIdentity === "hidden" ? "CACHÉE" : "RÉVÉLÉE"}</Text></View>
            <View style={styles.intenseMetaDivider} />
            <View><Text style={styles.intenseMetaLabel}>CHOIX</Text><Text style={styles.intenseMetaValue}>1 CARTE</Text></View>
          </View>
          <Text style={styles.intenseSectionLabel}>TOUCHE UNE CARTE</Text>
          <View style={styles.intenseCardGrid}>
            {cards.map((p, index) => <View key={p.id} style={styles.intenseCardWrap}>{renderIntenseCardBack(p, index)}</View>)}
          </View>
          {cards.length === 0 && <View style={styles.intenseWaitPanel}><Text style={styles.intenseWaitTitle}>Les cartes arrivent…</Text><Text style={styles.intenseWaitText}>Préparation des propositions.</Text></View>}
        </ScrollView>
      );
    }

    if (!isMyTurn) {
      return (
        <ScrollView contentContainerStyle={styles.scroll}>
          <View style={styles.waitCard}><Text style={styles.turnEmoji}>⏳</Text><Text style={styles.turnTitle}>{currentPlayer?.name}</Text><Text style={styles.waitText}>{t.waiting}</Text></View>
          <Button label={t.back} onPress={leaveGame} variant="outline" />
        </ScrollView>
      );
    }

    const cards = game.proposals;
    return (
      <ScrollView contentContainerStyle={styles.scroll}>
        <View style={[styles.redHero, { backgroundColor: liveTheme.hero }]}><Text style={styles.redHeroTitle}>{t.choose}</Text><Text style={styles.redHeroSub}>{t.chooseHelp}</Text></View>
        <View style={styles.turnMeta}><Text style={styles.turnMetaText}>Manche {game.round} / {game.maxRounds}</Text><Text style={styles.turnMetaText}>{game.roundIdentity === "hidden" ? "🙈" : "👁️"} Identité</Text></View>
        <View style={[styles.cardGrid, { backgroundColor: "transparent" }]}>{cards.map((p, index) => <CardBack key={p.id} type={p.type} index={index} onPress={() => dispatch({ type: "SELECT_PROPOSAL", playerId: myId, proposalId: p.id })} actionBackground={liveTheme.cardAction} actionBorder={liveTheme.primary} truthBackground={liveTheme.cardTruth} truthBorder={liveTheme.secondary} />)}</View>
        {cards.length === 0 && <View style={styles.waitCard}><Text style={styles.waitText}>Préparation des cartes…</Text></View>}
      </ScrollView>
    );
  }

  function renderReveal() {
    if (!game || !selectedProposal) return null;

    if (game.gameMode === "intense") {
      return (
        <ScrollView contentContainerStyle={styles.intenseScroll}>
          {renderIntenseHeader(game.revealed ? "VOILÀ LA VÉRITÉ" : "PRÊT À DÉCOUVRIR ?", game.revealed ? "Le secret est levé. À présent, il faut jouer le jeu." : "Tu as choisi. Maintenant, découvre ce qui se cache derrière.", game.revealed ? "RÉVÉLATION" : "SUSPENSE")}
          {!game.revealed ? (
            <>
              <View style={[styles.intenseSecretCard, selectedProposal.type === "action" ? styles.intenseSecretAction : styles.intenseSecretTruth]}>
                <View style={styles.intenseSecretRing}><Text style={styles.intenseSecretLetter}>{selectedProposal.type === "action" ? "A" : "V"}</Text></View>
                <Text style={styles.intenseSecretType}>{selectedProposal.type === "action" ? "ACTION" : "VÉRITÉ"}</Text>
                <Text style={styles.intenseSecretMystery}>Le contenu est encore caché</Text>
                <Text style={styles.intenseSecretDots}>•••</Text>
              </View>
              <TouchableOpacity activeOpacity={0.88} onPress={() => dispatch({ type: "REVEAL_PROPOSAL", playerId: myId })} style={styles.intenseRevealButton}>
                <Text style={styles.intenseRevealButtonText}>RÉVÉLER LA PROPOSITION</Text>
                <Text style={styles.intenseRevealButtonSub}>Le moment est venu.</Text>
              </TouchableOpacity>
            </>
          ) : (
            <>
              <View style={[styles.intenseRevealedCard, selectedProposal.type === "action" ? styles.intenseRevealedAction : styles.intenseRevealedTruth]}>
                <View style={styles.intenseRevealedTop}><Text style={styles.intenseRevealedBadge}>{selectedProposal.type === "action" ? "ACTION" : "VÉRITÉ"}</Text><Text style={styles.intenseRevealedMark}>✦</Text></View>
                <Text style={styles.intenseRevealedText}>{selectedProposal.text}</Text>
                <View style={styles.intenseRevealedFooter}>
                  <Text style={styles.intenseRevealedAuthor}>{game.roundIdentity === "hidden" ? "Identité cachée pendant ce tour" : `Proposé par ${game.players.find((p) => p.id === selectedProposal.authorId)?.name ?? "joueur"}`}</Text>
                </View>
              </View>
              <View style={styles.intenseExecutePanel}>
                <Text style={styles.intenseExecuteKicker}>{selectedProposal.type === "action" ? "À EXÉCUTER" : "À RÉPONDRE"}</Text>
                <Text style={styles.intenseExecuteTitle}>{selectedProposal.type === "action" ? "Ose. Fais-le." : "Pas d'échappatoire. Réponds."}</Text>
                <Text style={styles.intenseExecuteText}>{selectedProposal.type === "action" ? "La cible doit réaliser l'Action." : "La cible doit répondre à la Vérité."}</Text>
              </View>
              <TouchableOpacity activeOpacity={0.88} onPress={() => dispatch({ type: "NEXT_TURN", playerId: myId })} style={styles.intenseMainButton}>
                <Text style={styles.intenseMainButtonText}>TERMINER LA MANCHE</Text><Text style={styles.intenseMainButtonArrow}>→</Text>
              </TouchableOpacity>
            </>
          )}
        </ScrollView>
      );
    }

    return (
      <ScrollView contentContainerStyle={styles.scroll}>
        <Text style={styles.screenTitle}>{t.revealedContent}</Text>
        {game.gameMode === "target" && <Text style={styles.helper}>🎯 {currentPlayer?.name ?? "Joueur"} a ciblé {game.players.find((p) => p.id === game.targetPlayerId)?.name ?? "un joueur"}.</Text>}
        {!game.revealed ? (
          <>
            <View style={[styles.secretCard, { backgroundColor: liveTheme.text }]}><Text style={styles.secretEmoji}>{selectedProposal.type === "action" ? "🔥" : "💬"}</Text><Text style={styles.secretType}>{selectedProposal.type === "action" ? "ACTION" : "VÉRITÉ"}</Text><Text style={styles.secretText}>Le contenu est toujours caché.</Text></View>
            <Button label={t.reveal} onPress={() => dispatch({ type: "REVEAL_PROPOSAL", playerId: myId })} variant="pink" />
          </>
        ) : (
          <>
            <View style={[styles.revealedCard, { borderColor: liveTheme.primary, backgroundColor: liveTheme.surface }]}><Text style={styles.revealedType}>{selectedProposal.type === "action" ? "🔥 ACTION" : "💬 VÉRITÉ"}</Text><Text style={styles.revealedText}>{selectedProposal.text}</Text><Text style={styles.authorText}>{game.roundIdentity === "hidden" ? "🙈 Identité cachée pendant ce tour" : `Proposé par ${game.players.find((p) => p.id === selectedProposal.authorId)?.name ?? "joueur"}`}</Text></View>
            <View style={[styles.executeCard, { backgroundColor: liveTheme.accent }]}><Text style={styles.executeTitle}>{selectedProposal.type === "action" ? "🔥 À EXÉCUTER" : "💬 À RÉPONDRE"}</Text><Text style={styles.executeText}>{selectedProposal.type === "action" ? "La cible doit exécuter l'Action." : "La cible doit répondre à la Vérité."}</Text></View>
            <Button label={t.done} onPress={() => dispatch({ type: "NEXT_TURN", playerId: myId })} variant="pink" />
          </>
        )}
      </ScrollView>
    );
  }

  function renderPrivateComposer() {
    if (!game) return null;

    const others = game.players.filter((p) => p.id !== myId);
    if (!others.length) return null;

    if (!showPrivateComposer) {
      return (
        <TouchableOpacity
          onPress={() => { setShowPrivateComposer(true); setUnreadPrivateCount(0); }}
          style={[
            styles.privateLauncher,
            {
              backgroundColor: liveTheme.surface,
              borderColor: liveTheme.border,
            },
          ]}
          activeOpacity={0.85}
        >
          <View style={styles.privateIconWrap}>
            <Text style={styles.privateLauncherIcon}>💬</Text>
            {unreadPrivateCount > 0 && (
              <View style={styles.privateUnreadBadge}>
                <Text style={styles.privateUnreadText}>{unreadPrivateCount > 9 ? "9+" : unreadPrivateCount}</Text>
              </View>
            )}
          </View>
          <View style={{ flex: 1 }}>
            <Text style={styles.privateLauncherTitle}>{t.private}</Text>
            <Text style={styles.privateLauncherText}>Écrire discrètement à un joueur</Text>
          </View>
          <Text style={styles.privateLauncherArrow}>›</Text>
        </TouchableOpacity>
      );
    }

    return (
      <View style={[styles.privateBox, { backgroundColor: liveTheme.surface, borderColor: liveTheme.border }]}>
        <View style={styles.privateHeader}>
          <View>
            <Text style={styles.sectionTitle}>💬 {t.private}</Text>
            <Text style={styles.privateHint}>Conversation privée entre vous deux.</Text>
          </View>
          <TouchableOpacity
            onPress={() => setShowPrivateComposer(false)}
            style={[styles.privateCloseButton, game.gameMode === "intense" && styles.intensePrivateCloseButton]}
            activeOpacity={0.8}
          >
            <Text style={styles.privateCloseText}>✕</Text>
          </TouchableOpacity>
        </View>

        <ScrollView horizontal showsHorizontalScrollIndicator={false}>
          {others.map((p) => (
            <TouchableOpacity
              key={p.id}
              onPress={() => setPrivateTarget(p.id)}
              style={[
                styles.privatePlayer,
                privateTarget === p.id && styles.privatePlayerSelected,
              ]}
            >
              <Text>{p.avatar}</Text>
              <Text style={styles.privatePlayerText}>{p.name}</Text>
            </TouchableOpacity>
          ))}
        </ScrollView>

        {privateTarget && privateMessages.slice(-2).map((m) => (
          <View key={m.id} style={styles.privateMessageRow}>
            <Text style={styles.privateMessageLabel}>
              {m.fromId === myId
                ? "Moi"
                : (game.players.find((p) => p.id === m.fromId)?.name ?? "Joueur")}
            </Text>
            <Text style={styles.privateMessageText}>{m.text}</Text>
          </View>
        ))}

        {privateTarget && (
          <>
            <TextInput
              value={privateText}
              onChangeText={setPrivateText}
              placeholder={`Message privé → ${
                game.players.find((p) => p.id === privateTarget)?.name ?? ""
              }`}
              placeholderTextColor="#9D91A9"
              style={styles.input}
            />

            <Button
              label={t.sendMessage}
              onPress={() => {
                if (!privateText.trim()) return;
                sendPrivateMessage(privateTarget, privateText);
                setPrivateText("");
              }}
              variant="yellow"
            />
          </>
        )}
      </View>
    );
  }

  function renderDemaskingVote() {
    if (!game) return null;
    const authors: string[] = Array.from(new Set<string>(
      game.proposals.filter((p) => p.authorId !== game.currentPlayerId).map((p) => p.authorId)
    ));
    const hasVoted = !!game.demaskingVotes[myId];
    const currentPlayer = game.players.find((p) => p.id === game.currentPlayerId);

    return (
      <ScrollView contentContainerStyle={styles.intenseScroll}>
        {renderIntenseHeader("🕵️ TENTATIVE DE DÉMASQUAGE", "Qui a écrit la proposition choisie ?", "VOTE SECRET")}
        <View style={styles.intenseSpectator}>
          <Text style={styles.intenseSpectatorIcon}>?</Text>
          <Text style={styles.intenseSpectatorTitle}>À vous de deviner.</Text>
          <Text style={styles.intenseSpectatorText}>Chaque joueur vote une fois. La cible vote aussi. Les auteurs restent cachés jusqu'à la fin du vote.</Text>
        </View>
        {authors.map((authorId) => {
          const author = game.players.find((p) => p.id === authorId);
          return (
            <TouchableOpacity
              key={authorId}
              disabled={hasVoted}
              onPress={() => dispatch({ type: "SUBMIT_DEMASKING_VOTE", playerId: myId, authorId })}
              style={[styles.ruleCard, hasVoted && game.demaskingVotes[myId] === authorId && styles.ruleCardSelected, hasVoted && game.demaskingVotes[myId] !== authorId && styles.ruleCardDisabled]}
            >
              <Text style={styles.ruleTitle}>👤 {author?.name ?? "Joueur"}</Text>
              <Text style={styles.ruleText}>Je pense que c'est lui / elle.</Text>
            </TouchableOpacity>
          );
        })}
        {hasVoted ? (
          <View style={styles.waitCard}>
            <Text style={styles.waitText}>Vote enregistré. En attente des autres joueurs…</Text>
          </View>
        ) : null}
        <Text style={styles.helper}>Joueur en cours : {currentPlayer?.name ?? "Joueur"}</Text>
      </ScrollView>
    );
  }

  function renderDemaskingResult() {
    if (!game) return null;
    const winner = game.demaskingResult ? game.players.find((p) => p.id === game.demaskingResult) : null;
    const actualAuthor = selectedProposal ? game.players.find((p) => p.id === selectedProposal.authorId) : null;
    const isCurrentPlayer = game.currentPlayerId === myId;

    return (
      <ScrollView contentContainerStyle={styles.intenseScroll}>
        {renderIntenseHeader("🕵️ DÉMASQUAGE", "Le vote est terminé.", "RÉSULTAT")}
        <View style={styles.intenseRoundComplete}>
          <Text style={styles.intenseRoundNumber}>{game.demaskingTie ? "🤝" : "🎯"}</Text>
          <Text style={styles.intenseRoundCaption}>{game.demaskingTie ? "ÉGALITÉ" : "VOTE TERMINÉ"}</Text>
        </View>
        <View style={styles.intenseSpectator}>
          <Text style={styles.intenseSpectatorTitle}>Auteur réel</Text>
          <Text style={styles.intenseSpectatorText}>{actualAuthor?.name ?? "Joueur"}</Text>
          {game.demaskingTie ? (
            <Text style={styles.intenseSpectatorText}>Égalité : personne n'est démasqué par le vote.</Text>
          ) : (
            <Text style={styles.intenseSpectatorText}>Le vote majoritaire désigne : {winner?.name ?? "Joueur"}.</Text>
          )}
        </View>
        {isCurrentPlayer ? (
          <TouchableOpacity activeOpacity={0.88} onPress={() => dispatch({ type: "CONTINUE_AFTER_DEMASKING", playerId: myId })} style={styles.intenseMainButton}>
            <Text style={styles.intenseMainButtonText}>TOUR SUIVANT</Text><Text style={styles.intenseMainButtonArrow}>→</Text>
          </TouchableOpacity>
        ) : (
          <View style={styles.waitCard}><Text style={styles.waitText}>Le joueur en cours lance le tour suivant…</Text></View>
        )}
      </ScrollView>
    );
  }

  function renderRoundEnd() {
    if (!game) return null;

    if (game.gameMode === "intense") {
      return (
        <ScrollView contentContainerStyle={styles.intenseScroll}>
          {renderIntenseHeader("LA MANCHE EST TERMINÉE", "Le suspense retombe… avant que la prochaine manche recommence.", "ROUND COMPLETE")}
          <View style={styles.intenseRoundComplete}>
            <Text style={styles.intenseRoundNumber}>{game.round}</Text>
            <Text style={styles.intenseRoundCaption}>MANCHE TERMINÉE</Text>
          </View>
          {game.players.map((p) => (
            <View key={p.id} style={styles.intenseScoreRow}>
              <View style={styles.intenseScoreAvatar}><Text>{p.avatar}</Text></View>
              <Text style={styles.intenseScoreName}>{p.name}</Text>
              <Text style={styles.intenseScore}>{game.score[p.id] ?? 0}</Text>
            </View>
          ))}
          {game.hostId === myId && (
            <TouchableOpacity activeOpacity={0.88} onPress={() => dispatch({ type: "NEXT_ROUND", playerId: myId })} style={styles.intenseMainButton}>
              <Text style={styles.intenseMainButtonText}>{game.round >= game.maxRounds ? t.gameEnd : "LANCER LA PROCHAINE MANCHE"}</Text><Text style={styles.intenseMainButtonArrow}>→</Text>
            </TouchableOpacity>
          )}
        </ScrollView>
      );
    }

    return (
      <ScrollView contentContainerStyle={styles.scroll}>
        <View style={styles.goodCard}>
          <Text style={styles.goodEmoji}>🎉</Text>
          <Text style={styles.goodTitle}>{t.roundEnd}</Text>
          <Text style={styles.goodText}>
            La manche {game.round} est terminée.
          </Text>
        </View>

        {game.players.map((p) => (
          <View key={p.id} style={styles.scoreRow}>
            <Text style={styles.playerName}>
              {p.avatar} {p.name}
            </Text>
            <Text style={styles.score}>{game.score[p.id] ?? 0}</Text>
          </View>
        ))}


        {game.hostId === myId && (
          <Button
            label={game.round >= game.maxRounds ? t.gameEnd : t.next}
            onPress={() =>
              dispatch({
                type: "NEXT_ROUND",
                playerId: myId,
              })
            }
            variant="pink"
          />
        )}
      </ScrollView>
    );
  }

  function renderInGameSettings() {
    if (!game) return null;
    const isHost = game.hostId === myId;

    return (
      <Modal
        visible={showInGameSettings}
        transparent
        animationType="slide"
        onRequestClose={() => setShowInGameSettings(false)}
      >
        <View style={styles.settingsOverlay}>
          <View style={styles.settingsModal}>
            <View style={styles.settingsModalHeader}>
              <Text style={styles.settingsModalTitle}>⚙️ {t.settings}</Text>
              <TouchableOpacity onPress={() => setShowInGameSettings(false)} style={styles.closeButton}>
                <Text style={styles.closeButtonText}>✕</Text>
              </TouchableOpacity>
            </View>

            <ScrollView showsVerticalScrollIndicator={false} contentContainerStyle={{ paddingBottom: 10 }}>
              <View style={styles.roomPanel}>
                <Text style={styles.sectionTitle}>🏠 Salle</Text>
                <Text style={styles.roomCodeLabel}>Code de la salle</Text>
                <Text style={styles.roomCodeValue}>{game.code}</Text>
                <View style={styles.roomActionsRow}>
                  <Button label="📋 Copier" onPress={() => Clipboard.setStringAsync(game.code)} variant="outline" />
                  <Button label="📤 Partager" onPress={() => Share.share({ message: `Rejoins ma partie AVOU avec le code ${game.code}` })} variant="yellow" />
                </View>
                <Text style={styles.helper}>Le code reste disponible pendant toute la partie pour permettre à de nouveaux joueurs de rejoindre.</Text>
              </View>
              <Text style={styles.sectionTitle}>{t.gameMode}</Text>
              <TouchableOpacity
                disabled={!isHost}
                onPress={() => dispatch({ type: "UPDATE_GAME_SETTINGS", playerId: myId, identityMode: game.identityMode, gameMode: "target", orderMode: game.orderMode ?? "order" })}
                style={[styles.ruleCard, game.gameMode === "target" && styles.ruleCardSelected, !isHost && styles.ruleCardDisabled]}
              >
                <Text style={styles.ruleTitle}>{t.targetMode}</Text>
                <Text style={styles.ruleText}>{t.targetModeHelp}</Text>
              <Text style={styles.helper}>👥 2 joueurs minimum</Text>
              </TouchableOpacity>
              <TouchableOpacity
                disabled={!isHost}
                onPress={() => dispatch({ type: "UPDATE_GAME_SETTINGS", playerId: myId, identityMode: game.identityMode, gameMode: "intense", orderMode: game.orderMode ?? "order" })}
                style={[styles.ruleCard, game.gameMode === "intense" && styles.ruleCardSelected, !isHost && styles.ruleCardDisabled]}
              >
                <Text style={styles.ruleTitle}>{t.intenseMode}</Text>
                <Text style={styles.ruleText}>{t.intenseModeHelp}</Text>
              <Text style={styles.helper}>👥 3 joueurs minimum</Text>
              </TouchableOpacity>

              {game.gameMode === "intense" && (
                <>
                  <Text style={styles.sectionTitle}>🕵️ TENTATIVE DE DÉMASQUAGE</Text>
                  <TouchableOpacity
                    disabled={!isHost}
                    onPress={() => dispatch({ type: "SET_DEMASKING_ENABLED", playerId: myId, enabled: !game.demaskingEnabled })}
                    style={[styles.ruleCard, game.demaskingEnabled && styles.ruleCardSelected, !isHost && styles.ruleCardDisabled]}
                  >
                    <Text style={styles.ruleTitle}>{game.demaskingEnabled ? "🕵️ Activée" : "🕵️ Désactivée"}</Text>
                    <Text style={styles.ruleText}>Après chaque tour, tous les joueurs votent pour deviner qui a écrit la proposition choisie. La cible vote aussi.</Text>
                  </TouchableOpacity>
                </>
              )}

              <Text style={styles.sectionTitle}>{t.identities}</Text>
              {(
                [
                  ["reveal", t.revealed],
                  ["hidden", t.hidden],
                ] as [IdentityMode, string][]
              ).map(([mode, label]) => (
                <TouchableOpacity
                  key={mode}
                  disabled={!isHost}
                  onPress={() => dispatch({ type: "UPDATE_GAME_SETTINGS", playerId: myId, identityMode: mode, gameMode: game.gameMode, orderMode: game.orderMode ?? "order" })}
                  style={[styles.ruleCard, game.identityMode === mode && styles.ruleCardSelected, !isHost && styles.ruleCardDisabled]}
                >
                  <Text style={styles.ruleTitle}>{label}</Text>
                  <Text style={styles.ruleText}>
                    {mode === "reveal"
                      ? "Les identités sont visibles pendant les tours."
                      : "Les identités restent cachées pendant le tour."}
                  </Text>
                </TouchableOpacity>
              ))}

              <Text style={styles.sectionTitle}>ORDRE / DÉSORDRE</Text>
              <TouchableOpacity
                disabled={!isHost}
                onPress={() => dispatch({ type: "UPDATE_GAME_SETTINGS", playerId: myId, identityMode: game.identityMode, gameMode: game.gameMode, orderMode: "order" })}
                style={[styles.ruleCard, game.orderMode === "order" && styles.ruleCardSelected, !isHost && styles.ruleCardDisabled]}
              >
                <Text style={styles.ruleTitle}>🔢 {t.order}</Text>
                <Text style={styles.ruleText}>{t.orderHelp}</Text>
              </TouchableOpacity>
              <TouchableOpacity
                disabled={!isHost}
                onPress={() => dispatch({ type: "UPDATE_GAME_SETTINGS", playerId: myId, identityMode: game.identityMode, gameMode: game.gameMode, orderMode: "disorder" })}
                style={[styles.ruleCard, game.orderMode === "disorder" && styles.ruleCardSelected, !isHost && styles.ruleCardDisabled]}
              >
                <Text style={styles.ruleTitle}>🎲 {t.disorder}</Text>
                <Text style={styles.ruleText}>{t.disorderHelp}</Text>
              </TouchableOpacity>

              {isHost && game.joinRequests.length > 0 && (
                <View style={styles.requestsBox}>
                  <Text style={styles.sectionTitle}>👑 {t.pendingJoin}</Text>
                  {game.joinRequests.map((request) => (
                    <View key={request.id} style={styles.requestRow}>
                      <Text style={styles.playerName}>{request.avatar} {request.name}</Text>
                      <View style={styles.requestButtons}>
                        <Button label={t.accept} onPress={() => dispatch({ type: "APPROVE_JOIN", playerId: myId, requesterId: request.id })} variant="pink" />
                        <Button label={t.reject} onPress={() => dispatch({ type: "REJECT_JOIN", playerId: myId, requesterId: request.id })} variant="outline" />
                      </View>
                    </View>
                  ))}
                </View>
              )}

              {isHost && (
                <View style={styles.playerManagement}>
                  <Text style={styles.sectionTitle}>👥 Joueurs</Text>
                  <TextInput
                    value={addPlayerName}
                    onChangeText={setAddPlayerName}
                    placeholder="Pseudo du joueur à ajouter"
                    placeholderTextColor="#9D91A9"
                    style={styles.input}
                  />
                  <Button
                    label="➕ Ajouter directement"
                    onPress={() => {
                      if (!addPlayerName.trim()) return;
                      dispatch({ type: "ADD_PLAYER_DIRECT", playerId: myId, name: addPlayerName });
                      setAddPlayerName("");
                    }}
                    variant="yellow"
                  />
                  {game.players.filter((p) => p.id !== myId).map((p) => (
                    <View key={p.id} style={styles.managePlayerRow}>
                      <Text style={styles.playerName}>{p.avatar} {p.name}</Text>
                      <TouchableOpacity onPress={() => dispatch({ type: "REMOVE_PLAYER", playerId: myId, targetPlayerId: p.id })}>
                        <Text style={styles.removeText}>✕ {t.removePlayer}</Text>
                      </TouchableOpacity>
                    </View>
                  ))}
                </View>
              )}

              {!isHost && <Text style={styles.hostOnlyText}>👑 {t.hostOnly}</Text>}

              <Button label={t.leaveGame} onPress={requestLeaveGame} variant="yellow" />

              <Button
                label={t.closeSettings}
                onPress={() => setShowInGameSettings(false)}
                variant="outline"
              />
            </ScrollView>
          </View>
        </View>
      </Modal>
    );
  }

  function renderGameEnd() {
    if (!game) return null;

    return (
      <ScrollView contentContainerStyle={styles.scroll}>
        <View style={styles.goodCard}>
          <Text style={styles.goodEmoji}>🏆</Text>
          <Text style={styles.goodTitle}>{t.gameEnd}</Text>
          <Text style={styles.goodText}>Merci d'avoir joué à AVOU.</Text>
        </View>

        {game.players.map((p) => (
          <View key={p.id} style={styles.scoreRow}>
            <Text style={styles.playerName}>
              {p.avatar} {p.name}
            </Text>
            <Text style={styles.score}>{game.score[p.id] ?? 0}</Text>
          </View>
        ))}

        <Button label="Accueil" onPress={leaveGame} variant="pink" />
      </ScrollView>
    );
  }

  function renderGame() {
    if (!game) {
      return (
        <View style={styles.center}>
          <Text style={styles.screenTitle}>Connexion à la partie…</Text>
        </View>
      );
    }

    const isMember = game.players.some((p) => p.id === myId);
    const isPending = game.joinRequests.some((p) => p.id === myId);

    if (!isMember) {
      return (
        <View style={styles.center}>
          <View style={styles.goodCard}>
            <Text style={styles.goodEmoji}>⏳</Text>
            <Text style={styles.goodTitle}>{t.waitingApproval}</Text>
            <Text style={styles.goodText}>{isPending ? game.code : t.removed}</Text>
          </View>
          {!isPending && <Button label={t.back} onPress={() => { closeChannel(); setGame(null); setScreen("join"); }} variant="outline" />}
        </View>
      );
    }

    const canOpenSettings = game.phase !== "gameEnd";

    return (
      <View style={[styles.gameShell, { backgroundColor: liveTheme.background }]}>
        <View style={[styles.modeLiveStrip, { backgroundColor: liveTheme.hero }]}>
          <Text style={styles.modeLiveStripText}>
            {game.gameMode === "intense" ? "HOT VIBE  •  MODE INTENSE" : "GOOD VIBE  •  MODE CIBLE"}
          </Text>
        </View>

        {canOpenSettings && (
          <TouchableOpacity
            onPress={() => setShowInGameSettings(true)}
            style={[
              styles.inGameSettingsButton,
              {
                backgroundColor: liveTheme.surface,
                borderColor: liveTheme.border,
              },
            ]}
            activeOpacity={0.85}
          >
            <Text style={styles.inGameSettingsText}>⚙️</Text>
          </TouchableOpacity>
        )}
        {(() => {
          switch (game.phase) {
      case "lobby":
        return renderLobby();
      case "settings":
        return renderSettings();
      case "orderChoice":
        return renderOrderChoice();
      case "preparation":
        return renderPreparation();
      case "play":
        return renderPlay();
      case "reveal":
        return renderReveal();
      case "demaskingVote":
        return renderDemaskingVote();
      case "demaskingResult":
        return renderDemaskingResult();
      case "roundEnd":
        return renderRoundEnd();
      case "gameEnd":
        return renderGameEnd();
            default:
              return null;
          }
        })()}
        {renderPrivateComposer()}
        {renderInGameSettings()}
      </View>
    );
  }

  let content: React.ReactNode;

  switch (screen) {
    case "welcome":
      content = renderWelcome();
      break;
    case "home":
      content = renderHome();
      break;
    case "create":
      content = renderCreate();
      break;
    case "join":
      content = renderJoin();
      break;
    case "language":
      content = renderLanguage();
      break;
    case "avatar":
      content = renderAvatar();
      break;
    case "space":
      content = renderSpace();
      break;
    case "game":
      content = renderGame();
      break;
    default:
      content = renderWelcome();
  }

  return (
    <SafeAreaView style={styles.safe}>
      <StatusBar barStyle={game?.gameMode === "intense" ? "light-content" : "dark-content"} backgroundColor={game?.gameMode === "intense" ? "#210A12" : "#FFF7F4"} />
      <KeyboardAvoidingView
        style={styles.flex}
        behavior={Platform.OS === "ios" ? "padding" : undefined}
      >
        {content}
      </KeyboardAvoidingView>
    </SafeAreaView>
  );
}

const styles = StyleSheet.create({
  intenseScroll: { padding: 18, paddingBottom: 110, backgroundColor: "#210A12" },
  intenseHero: { minHeight: 205, borderRadius: 32, padding: 22, marginBottom: 18, overflow: "hidden", backgroundColor: "#A90F38", borderWidth: 1, borderColor: "#FF6B78", shadowColor: "#FF174F", shadowOpacity: 0.34, shadowRadius: 24, shadowOffset: { width: 0, height: 10 }, elevation: 10 },
  intenseHeroGlow: { position: "absolute", width: 210, height: 210, borderRadius: 105, backgroundColor: "#FF5B7A", opacity: 0.18, right: -70, top: -85 },
  intenseHeroTopRow: { flexDirection: "row", justifyContent: "space-between", alignItems: "center" },
  intensePill: { borderRadius: 20, paddingVertical: 7, paddingHorizontal: 12, backgroundColor: "rgba(255,255,255,0.16)", borderWidth: 1, borderColor: "rgba(255,255,255,0.25)" },
  intensePillText: { color: "#FFF7F8", fontSize: 10, fontWeight: "900", letterSpacing: 1.3 },
  intenseHeroSpark: { color: "#FFD2DC", fontSize: 28 },
  intenseHeroTitle: { color: "#FFFFFF", fontSize: 31, fontWeight: "900", letterSpacing: 0.4, marginTop: 28 },
  intenseHeroSub: { color: "#FFE6EC", fontSize: 15, lineHeight: 22, marginTop: 8, maxWidth: 330 },
  intenseTargetFocus: { backgroundColor: "#35101C", borderRadius: 28, padding: 26, alignItems: "center", borderWidth: 1, borderColor: "#7F2944", marginBottom: 14 },
  intenseTargetKicker: { color: "#FF9AAF", fontSize: 10, fontWeight: "900", letterSpacing: 1.6 },
  intenseTargetAvatar: { fontSize: 56, marginTop: 12 },
  intenseTargetName: { color: "#FFFFFF", fontSize: 27, fontWeight: "900", marginTop: 4 },
  intenseTargetHint: { color: "#CDA7B1", textAlign: "center", lineHeight: 19, marginTop: 8 },
  intenseWaitPanel: { backgroundColor: "#2D0E18", borderRadius: 24, padding: 20, borderWidth: 1, borderColor: "#652235", marginBottom: 14 },
  intenseWaitTitle: { color: "#FFFFFF", fontSize: 17, fontWeight: "900", marginBottom: 5 },
  intenseWaitText: { color: "#CFA9B4", lineHeight: 20 },
  intenseSubmittedPanel: { backgroundColor: "#35101C", borderRadius: 28, padding: 28, alignItems: "center", borderWidth: 1, borderColor: "#9D3152" },
  intenseSubmittedIcon: { width: 64, height: 64, borderRadius: 32, textAlign: "center", textAlignVertical: "center", backgroundColor: "#FF315D", color: "#FFFFFF", fontSize: 34, fontWeight: "900", overflow: "hidden" },
  intenseSubmittedTitle: { color: "#FFFFFF", fontSize: 20, fontWeight: "900", marginTop: 16 },
  intenseSubmittedText: { color: "#D4AAB5", textAlign: "center", marginTop: 7, lineHeight: 20 },
  intenseTargetMini: { flexDirection: "row", alignItems: "center", backgroundColor: "#35101C", borderRadius: 22, padding: 13, marginBottom: 16, borderWidth: 1, borderColor: "#70263F" },
  intenseTargetMiniAvatar: { fontSize: 30, marginRight: 12 },
  intenseTargetMiniLabel: { color: "#B77B89", fontSize: 9, fontWeight: "900", letterSpacing: 1.3 },
  intenseTargetMiniName: { color: "#FFFFFF", fontSize: 17, fontWeight: "900", marginTop: 2 },
  intenseTargetMiniSpark: { color: "#FF6B78", fontSize: 24 },
  intenseInputBlock: { backgroundColor: "#2E0D17", borderRadius: 26, padding: 16, marginBottom: 14, borderWidth: 1, borderColor: "#632337" },
  intenseInputHeading: { flexDirection: "row", alignItems: "center", marginBottom: 12 },
  intenseTypeBadge: { width: 46, height: 46, borderRadius: 16, alignItems: "center", justifyContent: "center", marginRight: 12 },
  intenseActionBadge: { backgroundColor: "#D81F4C" },
  intenseTruthBadge: { backgroundColor: "#FF713E" },
  intenseTypeBadgeText: { color: "#FFFFFF", fontSize: 21, fontWeight: "900" },
  intenseInputTitle: { color: "#FFFFFF", fontSize: 16, fontWeight: "900", letterSpacing: 1 },
  intenseInputSub: { color: "#B9808C", fontSize: 12, marginTop: 2 },
  intenseTextArea: { minHeight: 105, borderRadius: 19, backgroundColor: "#18070D", borderWidth: 1, borderColor: "#57202F", padding: 15, color: "#FFFFFF", textAlignVertical: "top", fontSize: 15, lineHeight: 21 },
  intenseMainButton: { minHeight: 60, borderRadius: 22, backgroundColor: "#FF315D", alignItems: "center", justifyContent: "center", paddingHorizontal: 18, marginTop: 8, flexDirection: "row", shadowColor: "#FF315D", shadowOpacity: 0.28, shadowRadius: 16, shadowOffset: { width: 0, height: 7 }, elevation: 7 },
  intenseMainButtonDisabled: { opacity: 0.42 },
  intenseMainButtonText: { color: "#FFFFFF", fontSize: 13, fontWeight: "900", letterSpacing: 1 },
  intenseMainButtonArrow: { color: "#FFFFFF", fontSize: 23, fontWeight: "900", marginLeft: 12 },
  intenseSpectator: { backgroundColor: "#35101C", borderRadius: 28, padding: 28, alignItems: "center", borderWidth: 1, borderColor: "#762A43" },
  intenseSpectatorIcon: { width: 70, height: 70, borderRadius: 35, textAlign: "center", textAlignVertical: "center", backgroundColor: "#FF315D", color: "#FFFFFF", fontSize: 34, overflow: "hidden" },
  intenseSpectatorTitle: { color: "#FFFFFF", fontSize: 23, fontWeight: "900", marginTop: 16 },
  intenseSpectatorText: { color: "#CBA5AF", textAlign: "center", lineHeight: 21, marginTop: 8 },
  intenseRoundPill: { alignSelf: "center", borderRadius: 20, backgroundColor: "#4A1425", borderWidth: 1, borderColor: "#833049", paddingVertical: 9, paddingHorizontal: 15, marginTop: 16 },
  intenseRoundPillText: { color: "#FFB2C0", fontSize: 10, fontWeight: "900", letterSpacing: 1.3 },
  intenseChoiceMeta: { flexDirection: "row", alignItems: "center", justifyContent: "space-around", backgroundColor: "#35101C", borderRadius: 22, paddingVertical: 14, marginBottom: 20, borderWidth: 1, borderColor: "#70263F" },
  intenseMetaLabel: { color: "#B77B89", fontSize: 8, fontWeight: "900", textAlign: "center", letterSpacing: 1.1 },
  intenseMetaValue: { color: "#FFFFFF", fontSize: 11, fontWeight: "900", textAlign: "center", marginTop: 3 },
  intenseMetaDivider: { width: 1, height: 28, backgroundColor: "#6A263C" },
  intenseSectionLabel: { color: "#FF9AAF", fontSize: 10, fontWeight: "900", letterSpacing: 1.6, marginBottom: 10 },
  intenseCardGrid: { flexDirection: "row", flexWrap: "wrap", justifyContent: "space-between" },
  intenseCardWrap: { width: "48.5%", marginBottom: 12 },
  intenseCard: { height: 225, borderRadius: 27, padding: 15, overflow: "hidden", borderWidth: 1, position: "relative", justifyContent: "space-between", shadowColor: "#FF315D", shadowOpacity: 0.18, shadowRadius: 13, shadowOffset: { width: 0, height: 6 }, elevation: 6 },
  intenseCardAction: { backgroundColor: "#B8143E", borderColor: "#F14E70" },
  intenseCardTruth: { backgroundColor: "#B84627", borderColor: "#FF8D69" },
  intenseCardShine: { position: "absolute", width: 120, height: 120, borderRadius: 60, backgroundColor: "#FFFFFF", opacity: 0.08, top: -45, right: -35 },
  intenseCardMini: { color: "#FFE6EB", fontSize: 8, fontWeight: "900", letterSpacing: 1.1 },
  intenseCardCenter: { alignItems: "center" },
  intenseCardIcon: { width: 64, height: 64, borderRadius: 22, backgroundColor: "rgba(255,255,255,0.16)", color: "#FFFFFF", fontSize: 30, fontWeight: "900", textAlign: "center", textAlignVertical: "center", overflow: "hidden" },
  intenseCardType: { color: "#FFFFFF", fontSize: 17, fontWeight: "900", letterSpacing: 1.2, marginTop: 10 },
  intenseCardHint: { color: "#FFE1E7", fontSize: 10, marginTop: 5, textAlign: "center" },
  intenseCardNumber: { color: "rgba(255,255,255,0.65)", fontSize: 10, fontWeight: "900", textAlign: "right" },
  intenseSecretCard: { minHeight: 350, borderRadius: 34, alignItems: "center", justifyContent: "center", padding: 25, borderWidth: 1, overflow: "hidden", marginBottom: 18 },
  intenseSecretAction: { backgroundColor: "#9E1038", borderColor: "#F55B78" },
  intenseSecretTruth: { backgroundColor: "#A74328", borderColor: "#FF9274" },
  intenseSecretRing: { width: 100, height: 100, borderRadius: 50, borderWidth: 2, borderColor: "rgba(255,255,255,0.45)", backgroundColor: "rgba(255,255,255,0.12)", alignItems: "center", justifyContent: "center" },
  intenseSecretLetter: { color: "#FFFFFF", fontSize: 48, fontWeight: "900" },
  intenseSecretType: { color: "#FFFFFF", fontSize: 25, fontWeight: "900", letterSpacing: 2, marginTop: 18 },
  intenseSecretMystery: { color: "#FFDCE3", marginTop: 8 },
  intenseSecretDots: { color: "#FFFFFF", fontSize: 25, letterSpacing: 8, marginTop: 18 },
  intenseRevealButton: { minHeight: 70, borderRadius: 23, backgroundColor: "#FFFFFF", alignItems: "center", justifyContent: "center", borderWidth: 1, borderColor: "#FF8EA2", shadowColor: "#FF315D", shadowOpacity: 0.25, shadowRadius: 15, shadowOffset: { width: 0, height: 7 }, elevation: 7 },
  intenseRevealButtonText: { color: "#A30F36", fontSize: 14, fontWeight: "900", letterSpacing: 1 },
  intenseRevealButtonSub: { color: "#A86D78", fontSize: 11, marginTop: 3 },
  intenseRevealedCard: { minHeight: 300, borderRadius: 32, padding: 23, borderWidth: 1, marginBottom: 16 },
  intenseRevealedAction: { backgroundColor: "#8D0D31", borderColor: "#F65D7B" },
  intenseRevealedTruth: { backgroundColor: "#9B3E26", borderColor: "#FF9576" },
  intenseRevealedTop: { flexDirection: "row", justifyContent: "space-between", alignItems: "center" },
  intenseRevealedBadge: { color: "#FFFFFF", fontSize: 11, fontWeight: "900", letterSpacing: 1.6, backgroundColor: "rgba(255,255,255,0.15)", paddingVertical: 7, paddingHorizontal: 10, borderRadius: 12 },
  intenseRevealedMark: { color: "#FFD1DB", fontSize: 23 },
  intenseRevealedText: { color: "#FFFFFF", fontSize: 24, lineHeight: 32, fontWeight: "800", marginTop: 35 },
  intenseRevealedFooter: { marginTop: 28, paddingTop: 13, borderTopWidth: 1, borderTopColor: "rgba(255,255,255,0.18)" },
  intenseRevealedAuthor: { color: "#FFD7DF", fontSize: 11, fontWeight: "700" },
  intenseExecutePanel: { backgroundColor: "#35101C", borderRadius: 25, padding: 20, borderWidth: 1, borderColor: "#7A2944", marginBottom: 14 },
  intenseExecuteKicker: { color: "#FF718A", fontSize: 9, fontWeight: "900", letterSpacing: 1.6 },
  intenseExecuteTitle: { color: "#FFFFFF", fontSize: 21, fontWeight: "900", marginTop: 7 },
  intenseExecuteText: { color: "#CDA8B2", lineHeight: 20, marginTop: 6 },
  intensePrivateBox: { borderWidth: 1, borderRadius: 24, padding: 14 },
  intensePrivateCloseButton: { backgroundColor: "#FF315D" },
  intenseRoundComplete: { backgroundColor: "#35101C", borderRadius: 28, padding: 24, alignItems: "center", marginBottom: 14, borderWidth: 1, borderColor: "#782A43" },
  intenseRoundNumber: { color: "#FF4E70", fontSize: 58, fontWeight: "900" },
  intenseRoundCaption: { color: "#D5A9B4", fontSize: 10, fontWeight: "900", letterSpacing: 1.6 },
  intenseScoreRow: { flexDirection: "row", alignItems: "center", backgroundColor: "#2D0E18", borderRadius: 18, padding: 12, marginBottom: 8, borderWidth: 1, borderColor: "#5E2034" },
  intenseScoreAvatar: { width: 42, height: 42, borderRadius: 21, backgroundColor: "#4A1425", alignItems: "center", justifyContent: "center", marginRight: 11 },
  intenseScoreName: { flex: 1, color: "#FFFFFF", fontWeight: "800" },
  intenseScore: { color: "#FF718A", fontSize: 20, fontWeight: "900" },

  turnEmoji: {
    fontSize: 32,
    textAlign: "center",
    marginBottom: 8,
  },

  turnTile: {
    backgroundColor: "#FFFFFF",
    borderRadius: 20,
    padding: 18,
    marginBottom: 12,
    borderWidth: 1,
    borderColor: "#E9D5FF",
    alignItems: "center",
  },

  turnTitle: {
    fontSize: 20,
    fontWeight: "800",
    color: "#35151D",
    textAlign: "center",
    marginBottom: 8,
  },

  flex: { flex: 1 },

  safe: {
    flex: 1,
    backgroundColor: "#FFF7F4",
  },

  center: {
    flex: 1,
    padding: 24,
    justifyContent: "center",
    backgroundColor: "#FFF7F4",
  },

  gameShell: {
    flex: 1,
    backgroundColor: "#FFF7F4",
  },

  modeLiveStrip: {
    marginHorizontal: 14,
    marginTop: 8,
    marginBottom: 4,
    borderRadius: 16,
    paddingVertical: 9,
    paddingHorizontal: 14,
    alignItems: "center",
  },

  modeLiveStripText: {
    color: "#FFFFFF",
    fontSize: 12,
    fontWeight: "900",
    letterSpacing: 1,
  },

  modeThemeBanner: {
    borderRadius: 24,
    padding: 20,
    marginBottom: 16,
  },

  modeThemeBannerTitle: {
    color: "#FFFFFF",
    fontSize: 19,
    fontWeight: "900",
  },

  modeThemeBannerText: {
    color: "#FFFFFF",
    opacity: 0.92,
    lineHeight: 21,
    marginTop: 7,
  },

  inGameSettingsButton: {
    position: "absolute",
    top: 12,
    right: 16,
    zIndex: 50,
    width: 48,
    height: 48,
    borderRadius: 24,
    backgroundColor: "#FFFFFF",
    alignItems: "center",
    justifyContent: "center",
    borderWidth: 2,
    borderColor: "#E9284F",
    shadowColor: "#E9284F",
    shadowOpacity: 0.18,
    shadowRadius: 10,
    shadowOffset: { width: 0, height: 4 },
    elevation: 8,
  },

  inGameSettingsText: {
    fontSize: 23,
  },

  settingsOverlay: {
    flex: 1,
    backgroundColor: "rgba(37, 27, 46, 0.45)",
    justifyContent: "flex-end",
  },

  settingsModal: {
    maxHeight: "92%",
    backgroundColor: "#FFF7F4",
    borderTopLeftRadius: 30,
    borderTopRightRadius: 30,
    padding: 20,
  },

  settingsModalHeader: {
    flexDirection: "row",
    alignItems: "center",
    justifyContent: "space-between",
    marginBottom: 8,
  },

  settingsModalTitle: {
    fontSize: 23,
    fontWeight: "900",
    color: "#251B2E",
  },

  closeButton: {
    width: 42,
    height: 42,
    borderRadius: 21,
    backgroundColor: "#FFE36E",
    alignItems: "center",
    justifyContent: "center",
  },

  closeButtonText: {
    fontSize: 19,
    fontWeight: "900",
    color: "#251B2E",
  },

  ruleCardDisabled: {
    opacity: 0.55,
  },

  hostOnlyText: {
    color: "#75687F",
    fontWeight: "700",
    lineHeight: 20,
    marginVertical: 8,
  },

  scroll: {
    padding: 20,
    paddingBottom: 70,
    backgroundColor: "#FFF7F4",
  },

  logo: {
    width: 145,
    height: 145,
    borderRadius: 72,
    backgroundColor: "#E9284F",
    alignSelf: "center",
    alignItems: "center",
    justifyContent: "center",
    transform: [{ rotate: "-5deg" }],
    marginBottom: 20,
    shadowColor: "#E9284F",
    shadowOpacity: 0.25,
    shadowRadius: 20,
    shadowOffset: { width: 0, height: 10 },
    elevation: 8,
  },

  logoText: {
    color: "#FFFFFF",
    fontSize: 38,
    fontWeight: "900",
    letterSpacing: 3,
  },

  heroTitle: {
    textAlign: "center",
    fontSize: 30,
    fontWeight: "900",
    color: "#251B2E",
  },

  heroSub: {
    textAlign: "center",
    color: "#75687F",
    fontSize: 15,
    marginTop: 8,
    marginBottom: 28,
  },

  header: {
    flexDirection: "row",
    justifyContent: "space-between",
    alignItems: "center",
    marginBottom: 18,
  },

  brand: {
    fontSize: 32,
    fontWeight: "900",
    color: "#E9284F",
    letterSpacing: 2,
  },

  headerSub: {
    color: "#75687F",
    fontWeight: "700",
    marginTop: 2,
  },

  avatarSmall: {
    width: 48,
    height: 48,
    borderRadius: 24,
    backgroundColor: "#FFE36E",
    justifyContent: "center",
    alignItems: "center",
  },

  redHero: {
    backgroundColor: "#E9284F",
    borderRadius: 28,
    padding: 24,
    marginBottom: 18,
    shadowColor: "#E9284F",
    shadowOpacity: 0.2,
    shadowRadius: 16,
    shadowOffset: { width: 0, height: 8 },
    elevation: 6,
  },

  redHeroTitle: {
    color: "#FFFFFF",
    fontSize: 30,
    fontWeight: "900",
    letterSpacing: 1,
  },

  redHeroSub: {
    color: "#FFE9EF",
    fontSize: 14,
    lineHeight: 20,
    marginTop: 8,
  },

  targetGrid: {
    flexDirection: "row",
    flexWrap: "wrap",
    gap: 10,
    marginBottom: 12,
  },
  targetChoice: {
    width: "30%",
    minWidth: 95,
    backgroundColor: "#FFFFFF",
    borderRadius: 20,
    padding: 14,
    alignItems: "center",
    borderWidth: 2,
    borderColor: "#F0E3EA",
  },
  targetChoiceSelected: {
    borderColor: "#E9284F",
    backgroundColor: "#FFF0F3",
  },
  targetChoiceAvatar: { fontSize: 30 },
  targetChoiceName: {
    marginTop: 6,
    fontWeight: "900",
    color: "#2D2335",
  },
  typeRow: {
    flexDirection: "row",
    gap: 12,
    marginBottom: 8,
  },
  typeChoice: {
    flex: 1,
    backgroundColor: "#FFFFFF",
    borderRadius: 20,
    padding: 16,
    alignItems: "center",
    borderWidth: 2,
    borderColor: "#F0E3EA",
  },
  typeChoiceSelected: {
    borderColor: "#E9284F",
    backgroundColor: "#FFF0F3",
  },
  typeChoiceEmoji: { fontSize: 28 },
  typeChoiceText: {
    marginTop: 6,
    fontWeight: "900",
    color: "#2D2335",
  },

  button: {
    minHeight: 54,
    borderRadius: 18,
    backgroundColor: "#7C4DFF",
    alignItems: "center",
    justifyContent: "center",
    paddingHorizontal: 18,
    marginTop: 12,
  },

  buttonPink: {
    backgroundColor: "#E9284F",
  },

  buttonYellow: {
    backgroundColor: "#FFE36E",
  },

  buttonOutline: {
    backgroundColor: "#FFFFFF",
    borderWidth: 2,
    borderColor: "#E9DDE7",
  },

  buttonDisabled: {
    opacity: 0.4,
  },

  buttonText: {
    color: "#FFFFFF",
    fontSize: 16,
    fontWeight: "900",
  },

  buttonOutlineText: {
    color: "#2D2335",
  },

  language: {
    alignSelf: "center",
    marginTop: 18,
    paddingVertical: 10,
    paddingHorizontal: 16,
    borderRadius: 20,
    backgroundColor: "#FFFFFF",
  },

  languageText: {
    color: "#4D4058",
    fontWeight: "800",
  },

  screenTitle: {
    fontSize: 28,
    fontWeight: "900",
    color: "#251B2E",
    marginBottom: 20,
  },

  sectionTitle: {
    fontSize: 19,
    fontWeight: "900",
    color: "#251B2E",
    marginTop: 18,
    marginBottom: 10,
  },

  label: {
    fontSize: 14,
    fontWeight: "900",
    color: "#4D4058",
    marginTop: 14,
    marginBottom: 7,
  },

  helper: {
    color: "#75687F",
    lineHeight: 21,
    marginBottom: 10,
  },

  input: {
    backgroundColor: "#FFFFFF",
    borderRadius: 16,
    minHeight: 52,
    paddingHorizontal: 16,
    paddingVertical: 13,
    color: "#251B2E",
    borderWidth: 1,
    borderColor: "#E9DDE7",
    fontSize: 16,
  },

  textArea: {
    minHeight: 120,
    textAlignVertical: "top",
  },

  codeInput: {
    textAlign: "center",
    fontSize: 28,
    fontWeight: "900",
    letterSpacing: 7,
  },

  error: {
    color: "#C7183D",
    fontWeight: "800",
    textAlign: "center",
    marginTop: 12,
  },

  avatarGrid: {
    flexDirection: "row",
    flexWrap: "wrap",
    gap: 10,
    marginBottom: 12,
  },

  avatarChoice: {
    width: 58,
    height: 58,
    borderRadius: 20,
    backgroundColor: "#FFFFFF",
    alignItems: "center",
    justifyContent: "center",
    borderWidth: 2,
    borderColor: "#E9DDE7",
  },

  avatarChoiceSelected: {
    borderColor: "#E9284F",
    backgroundColor: "#FFE9EF",
  },

  avatarEmoji: {
    fontSize: 29,
  },

  avatarBig: {
    width: 130,
    height: 130,
    borderRadius: 65,
    alignSelf: "center",
    alignItems: "center",
    justifyContent: "center",
    backgroundColor: "#FFE36E",
    marginBottom: 24,
  },

  avatarBigText: {
    fontSize: 65,
  },

  spaceCard: {
    backgroundColor: "#FFFFFF",
    borderRadius: 24,
    padding: 24,
    alignItems: "center",
    marginBottom: 18,
  },

  spaceAvatar: {
    fontSize: 70,
  },

  spaceName: {
    fontSize: 24,
    fontWeight: "900",
    color: "#251B2E",
    marginTop: 8,
  },

  spaceSub: {
    color: "#75687F",
    marginTop: 6,
  },

  codeCard: {
    backgroundColor: "#E9284F",
    borderRadius: 25,
    padding: 22,
    alignItems: "center",
    marginBottom: 12,
  },

  codeLabel: {
    color: "#FFE9EF",
    fontWeight: "800",
  },

  bigCode: {
    color: "#FFFFFF",
    fontSize: 42,
    fontWeight: "900",
    letterSpacing: 9,
    marginTop: 6,
  },

  requestsBox: {
    backgroundColor: "#FFF0F3",
    borderRadius: 22,
    padding: 16,
    marginBottom: 16,
    borderWidth: 1,
    borderColor: "#F2A2B4",
  },
  requestRow: {
    backgroundColor: "#FFFFFF",
    borderRadius: 16,
    padding: 12,
    marginBottom: 10,
  },
  requestButtons: {
    flexDirection: "row",
    gap: 8,
    marginTop: 8,
  },
  removeText: {
    color: "#E9284F",
    fontWeight: "900",
    paddingVertical: 8,
    paddingHorizontal: 6,
  },
  playerManagement: { marginTop: 10 },
  managePlayerRow: {
    backgroundColor: "#FFFFFF",
    borderRadius: 16,
    padding: 12,
    marginBottom: 8,
    flexDirection: "row",
    justifyContent: "space-between",
    alignItems: "center",
  },

  playerRow: {
    flexDirection: "row",
    alignItems: "center",
    backgroundColor: "#FFFFFF",
    borderRadius: 18,
    padding: 13,
    marginBottom: 8,
  },

  playerAvatar: {
    fontSize: 28,
    marginRight: 12,
  },

  playerInfo: {
    flex: 1,
  },

  playerName: {
    color: "#251B2E",
    fontSize: 16,
    fontWeight: "800",
  },

  hostBadge: {
    color: "#E9284F",
    fontSize: 12,
    fontWeight: "900",
    marginTop: 3,
  },

  waitCard: {
    backgroundColor: "#FFFFFF",
    borderRadius: 20,
    padding: 20,
    marginTop: 14,
    alignItems: "center",
  },

  waitText: {
    color: "#75687F",
    fontSize: 16,
    fontWeight: "700",
    textAlign: "center",
    lineHeight: 23,
  },

  ruleCard: {
    backgroundColor: "#FFFFFF",
    borderRadius: 22,
    padding: 19,
    marginBottom: 12,
    borderWidth: 2,
    borderColor: "#E9DDE7",
  },

  ruleCardSelected: {
    borderColor: "#E9284F",
    backgroundColor: "#FFF0F3",
  },

  ruleTitle: {
    color: "#251B2E",
    fontSize: 17,
    fontWeight: "900",
  },

  ruleText: {
    color: "#75687F",
    lineHeight: 21,
    marginTop: 6,
  },

  hostChoiceBox: {
    backgroundColor: "#FFE36E",
    borderRadius: 22,
    padding: 16,
    marginTop: 8,
  },

  hostChoiceTitle: {
    color: "#251B2E",
    fontWeight: "900",
    fontSize: 17,
    marginBottom: 6,
  },

  targetCard: {
    backgroundColor: "#7C4DFF",
    borderRadius: 25,
    padding: 22,
    alignItems: "center",
    marginBottom: 15,
  },

  targetEmoji: {
    fontSize: 48,
  },

  targetTitle: {
    color: "#FFFFFF",
    fontSize: 24,
    fontWeight: "900",
    marginTop: 5,
  },

  targetSub: {
    color: "#EEE6FF",
    fontWeight: "800",
  },

  progressCard: {
    backgroundColor: "#FFE36E",
    borderRadius: 22,
    padding: 18,
    alignItems: "center",
    marginVertical: 12,
  },

  progressTitle: {
    fontSize: 30,
    fontWeight: "900",
    color: "#251B2E",
  },

  progressText: {
    color: "#665B39",
    fontWeight: "800",
  },

  turnMeta: {
    flexDirection: "row",
    justifyContent: "space-between",
    marginBottom: 12,
  },

  turnMetaText: {
    color: "#75687F",
    fontWeight: "800",
  },

  cardGrid: {
    flexDirection: "row",
    flexWrap: "wrap",
    justifyContent: "space-between",
  },

  cardBack: {
    width: "47.5%",
    minHeight: 190,
    borderRadius: 24,
    padding: 16,
    marginBottom: 14,
    alignItems: "center",
    justifyContent: "center",
    borderWidth: 3,
  },

  actionBack: {
    backgroundColor: "#FFF0F3",
    borderColor: "#E9284F",
  },

  truthBack: {
    backgroundColor: "#F0E9FF",
    borderColor: "#7C4DFF",
  },

  cardIndex: {
    position: "absolute",
    top: 12,
    left: 14,
    color: "#75687F",
    fontWeight: "900",
  },

  cardBackEmoji: {
    fontSize: 43,
  },

  cardBackType: {
    fontSize: 18,
    fontWeight: "900",
    color: "#251B2E",
    marginTop: 8,
  },

  cardBackHidden: {
    fontSize: 10,
    color: "#75687F",
    fontWeight: "900",
    marginTop: 6,
  },

  secretCard: {
    backgroundColor: "#251B2E",
    borderRadius: 26,
    padding: 30,
    minHeight: 250,
    alignItems: "center",
    justifyContent: "center",
    marginBottom: 18,
  },

  secretEmoji: {
    fontSize: 58,
  },

  secretType: {
    color: "#FFFFFF",
    fontSize: 25,
    fontWeight: "900",
    marginTop: 12,
  },

  secretText: {
    color: "#DCD3E2",
    textAlign: "center",
    marginTop: 8,
  },

  revealedCard: {
    backgroundColor: "#FFFFFF",
    borderRadius: 26,
    padding: 25,
    borderWidth: 3,
    borderColor: "#E9284F",
  },

  revealedType: {
    color: "#E9284F",
    fontSize: 18,
    fontWeight: "900",
  },

  revealedText: {
    color: "#251B2E",
    fontSize: 23,
    lineHeight: 31,
    fontWeight: "900",
    marginTop: 14,
  },

  authorText: {
    color: "#75687F",
    marginTop: 18,
    fontWeight: "700",
  },

  executeCard: {
    backgroundColor: "#FFE36E",
    borderRadius: 20,
    padding: 18,
    marginTop: 14,
  },

  executeTitle: {
    color: "#251B2E",
    fontSize: 18,
    fontWeight: "900",
  },

  executeText: {
    color: "#665B39",
    marginTop: 6,
    lineHeight: 20,
  },

  modesInfoCard: { backgroundColor: "#FFF8D8", borderRadius: 22, padding: 16, marginBottom: 16, borderWidth: 1, borderColor: "#F1D66A" },
  modeInfoTitle: { fontSize: 15, fontWeight: "900", color: "#9E2C55", marginTop: 8 },
  modeInfoText: { fontSize: 13, lineHeight: 19, color: "#5F5362", marginTop: 3 },
  privateLauncher: {
    flexDirection: "row",
    alignItems: "center",
    borderWidth: 1,
    borderRadius: 18,
    paddingVertical: 11,
    paddingHorizontal: 14,
    marginHorizontal: 14,
    marginBottom: 10,
  },
  privateIconWrap: { position: "relative", width: 30, marginRight: 10, alignItems: "center" },
  privateLauncherIcon: { fontSize: 20 },
  privateUnreadBadge: { position: "absolute", top: -7, right: -2, minWidth: 18, height: 18, borderRadius: 9, backgroundColor: "#E9284F", alignItems: "center", justifyContent: "center", paddingHorizontal: 4 },
  privateUnreadText: { color: "#FFFFFF", fontSize: 10, fontWeight: "900" },
  privateLauncherTitle: { fontSize: 14, fontWeight: "900", color: "#3E3444" },
  privateLauncherText: { fontSize: 11, color: "#8B7A92", marginTop: 2 },
  privateLauncherArrow: { fontSize: 24, color: "#8B5AA8", marginLeft: 8 },
  privateHeader: {
    flexDirection: "row",
    alignItems: "center",
    justifyContent: "space-between",
    marginBottom: 8,
  },
  privateHint: { fontSize: 11, color: "#8B7A92", marginTop: -8, marginBottom: 8 },
  privateCloseButton: {
    width: 32,
    height: 32,
    borderRadius: 16,
    alignItems: "center",
    justifyContent: "center",
    backgroundColor: "#F5ECFF",
  },
  privateCloseText: { fontSize: 15, fontWeight: "800", color: "#6D4C7D" },
  privateMessageRow: { backgroundColor: "#F5ECFF", borderRadius: 14, padding: 10, marginTop: 7 },
  privateMessageLabel: { fontSize: 11, fontWeight: "800", color: "#8B5AA8" },
  privateMessageText: { fontSize: 14, color: "#3E3444", marginTop: 2 },
  roomPanel: {
    backgroundColor: "#FFF0F6",
    borderRadius: 22,
    padding: 16,
    marginBottom: 16,
    borderWidth: 1,
    borderColor: "#F3C4D8",
  },
  roomCodeLabel: { fontSize: 12, color: "#8A7280", fontWeight: "700", textTransform: "uppercase", letterSpacing: 1 },
  roomCodeValue: { fontSize: 30, fontWeight: "900", color: "#C51F55", letterSpacing: 5, textAlign: "center", marginVertical: 8 },
  roomActionsRow: { flexDirection: "row", gap: 8, marginBottom: 4 },
  privateBox: {
    backgroundColor: "#FFFFFF",
    borderRadius: 22,
    padding: 16,
    marginTop: 18,
  },

  privatePlayer: {
    backgroundColor: "#F5F0F7",
    borderRadius: 16,
    paddingVertical: 10,
    paddingHorizontal: 13,
    marginRight: 8,
    flexDirection: "row",
    alignItems: "center",
    gap: 5,
  },

  privatePlayerSelected: {
    backgroundColor: "#FFE9EF",
    borderWidth: 1,
    borderColor: "#E9284F",
  },

  privatePlayerText: {
    color: "#251B2E",
    fontWeight: "800",
  },

  languageRow: {
    backgroundColor: "#FFFFFF",
    padding: 18,
    borderRadius: 18,
    marginBottom: 10,
  },

  languageRowSelected: {
    borderWidth: 2,
    borderColor: "#E9284F",
    backgroundColor: "#FFF0F3",
  },

  languageRowText: {
    color: "#251B2E",
    fontSize: 17,
    fontWeight: "800",
  },

  goodCard: {
    backgroundColor: "#FFE36E",
    borderRadius: 28,
    padding: 26,
    alignItems: "center",
    marginBottom: 18,
  },

  goodEmoji: {
    fontSize: 56,
  },

  goodTitle: {
    color: "#251B2E",
    fontSize: 27,
    fontWeight: "900",
    marginTop: 8,
  },

  goodText: {
    color: "#665B39",
    textAlign: "center",
    marginTop: 6,
  },

  scoreRow: {
    backgroundColor: "#FFFFFF",
    borderRadius: 18,
    padding: 16,
    marginBottom: 8,
    flexDirection: "row",
    justifyContent: "space-between",
    alignItems: "center",
  },

  score: {
    color: "#E9284F",
    fontSize: 22,
    fontWeight: "900",
  },
});
