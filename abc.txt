import {
  setUserInfo,
  callJspringLogout,
  callPingfedLogout,
  logoutUser,
  authKBAccessAndRedirect,
} from './your-action-file'; // Change to your actual file path
import { SET_USER_INFO } from './action-types';
import {
  RefreshSessionService,
  CallJspringLogoutService,
  KBAuthService,
} from '../../services/user';
import { getLogoutURL } from '../../components/ping-fed-login/utils';

// Mock services and utils
jest.mock('../../services/user', () => ({
  RefreshSessionService: jest.fn(),
  CallJspringLogoutService: jest.fn(),
  KBAuthService: jest.fn(),
}));

jest.mock('../../components/ping-fed-login/utils', () => ({
  getLogoutURL: jest.fn(),
}));

describe('User Actions', () => {

  beforeEach(() => {
    // Reset mocks before each test
    jest.clearAllMocks();
  });

  describe('setUserInfo', () => {
    it('should return correct action type and data', () => {
      const data = { name: 'John' };
      const result = setUserInfo(data);
      
      expect(result).toEqual({
        type: SET_USER_INFO,
        data,
      });
    });
  });

  describe('callJspringLogout', () => {
    it('should resolve with response from CallJspringLogoutService', async () => {
      const mockResponse = { success: true };
      (CallJspringLogoutService as jest.Mock).mockResolvedValue(mockResponse);

      const result = await callJspringLogout();
      expect(result).toEqual(mockResponse);
      expect(CallJspringLogoutService).toHaveBeenCalledTimes(1);
    });

    it('should resolve with error on failure', async () => {
      const mockError = new Error('Logout failed');
      (CallJspringLogoutService as jest.Mock).mockRejectedValue(mockError);

      const result = await callJspringLogout();
      expect(result).toEqual(mockError);
    });
  });

  describe('callPingfedLogout', () => {
    it('should redirect to the PingFed logout URL and resolve', async () => {
      const mockLogoutURL = 'http://logout-url.com';
      (getLogoutURL as jest.Mock).mockReturnValue(mockLogoutURL);

      const windowLocationSpy = jest.spyOn(window, 'location', 'get');
      windowLocationSpy.mockReturnValue({ href: '' } as any);

      await callPingfedLogout();

      expect(getLogoutURL).toHaveBeenCalledWith(window.location.href);
      expect(window.location.href).toEqual(mockLogoutURL);
    });
  });

  describe('logoutUser', () => {
    it('should clear localStorage, sessionStorage and call services', async () => {
      (RefreshSessionService as jest.Mock).mockResolvedValue({});
      (CallJspringLogoutService as jest.Mock).mockResolvedValue({});
      
      const localStorageClearSpy = jest.spyOn(window.localStorage, 'clear');
      const sessionStorageClearSpy = jest.spyOn(window.sessionStorage, 'clear');

      await logoutUser();

      expect(localStorageClearSpy).toHaveBeenCalledTimes(1);
      expect(sessionStorageClearSpy).toHaveBeenCalledTimes(1);
      expect(RefreshSessionService).toHaveBeenCalledTimes(1);
      expect(CallJspringLogoutService).toHaveBeenCalledTimes(1);
      expect(callPingfedLogout).toHaveBeenCalledTimes(1);
    });

    it('should reject on RefreshSessionService failure', async () => {
      const mockError = new Error('Session refresh failed');
      (RefreshSessionService as jest.Mock).mockRejectedValue(mockError);

      await expect(logoutUser()).rejects.toThrow(mockError);
    });
  });

  describe('authKBAccessAndRedirect', () => {
    it('should call KBAuthService and submit form with correct data', async () => {
      const mockToken = 'mockToken';
      const mockKbAuthUrl = 'http://kb-auth-url.com';
      (KBAuthService as jest.Mock).mockResolvedValue({ token: mockToken });

      jest.spyOn(window.localStorage, 'getItem')
        .mockReturnValueOnce(mockKbAuthUrl) // For kbAuthUrl
        .mockReturnValueOnce('mockUser') // For userName
        .mockReturnValueOnce('mockWorkspaces'); // For kbWorkspaces

      const submitFORMSpy = jest.spyOn(document, 'createElement');

      await authKBAccessAndRedirect();

      expect(KBAuthService).toHaveBeenCalledTimes(1);
      expect(submitFORMSpy).toHaveBeenCalled();
    });

    it('should reject on KBAuthService failure', async () => {
      const mockError = new Error('KBAuth failed');
      (KBAuthService as jest.Mock).mockRejectedValue(mockError);

      await expect(authKBAccessAndRedirect()).rejects.toThrow(mockError);
    });
  });
});
